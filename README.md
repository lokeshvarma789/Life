
/* ============================================================================
   STEP 6 - CT TABLE LOAD DISTRIBUTION ANALYSIS  (ONE-SHOT, READ-ONLY)
   ----------------------------------------------------------------------------
   Analyzes ALL CT tables in a schema in ONE execution.
   No CREATE statements. No paste. Read-only access required.
   ----------------------------------------------------------------------------

   HOW IT WORKS (under the hood):
     1. Snowflake Scripting block (BEGIN ... END) runs as an anonymous block
     2. Builds a dynamic UNION ALL by reading INFORMATION_SCHEMA.TABLES
     3. EXECUTE IMMEDIATE runs that dynamic SQL and returns the result set
     4. No temp tables, no procedures, no privileges needed beyond SELECT

   FOUR DETECTION METHODS:
     1. PERCENTILE     : top 10% / bottom 10% within the schema
     2. STDDEV         : table avg vs schema-wide avg (2-stddev rule)
     3. IQR            : Q1 - 1.5*IQR / Q3 + 1.5*IQR (outlier detection)
     4. ABSOLUTE BANDS : dormant / very-low / very-high thresholds

   QLIK CDC HANDLING:
     BEFOREIMAGE rows are EXCLUDED from business counts.
     CDC_INTEGRITY_FLAG validates BEFOREIMAGE = UPDATE per table.

   PERFORMANCE:
     - Each CT table scanned ONCE (single pass)
     - HEADER__TIMESTAMP filter triggers micro-partition pruning
     - Snowflake result cache returns same-day repeat runs in seconds
     - Estimated runtime: 30 sec - 2 min depending on schema size

   AVAILABLE SCHEMAS : ING, FRATDB, RPLUS (Qlik) | L70, DCLM, SAP (Talend)

   HOW TO USE:
     1. Change the SCHEMA_NAME value in the SET line below
     2. Highlight the ENTIRE script
     3. Run -> results appear

   AUDIT TRAIL:
     Author : Lokesh Varma
     Date   : ____________
     Step   : 6 of monitoring suite
============================================================================ */

USE ROLE SF_PROD_DATA_ENGINEER;
USE WAREHOUSE PROD_DATA_ENGINEER_WH;
USE DATABASE PROD_LANDING;


/* ============================================================================
   ANONYMOUS BLOCK - RUNS AS ONE EXECUTION
============================================================================ */

DECLARE
    target_schema   STRING DEFAULT 'ING';                  -- > CHANGE SCHEMA HERE
    union_sql       STRING;
    full_query      STRING;
    res             RESULTSET;
BEGIN

    -- ------------------------------------------------------------------------
    -- Step A: Build dynamic UNION ALL across all CT tables in the schema
    -- ------------------------------------------------------------------------
    SELECT LISTAGG(
        'SELECT ''' || TABLE_SCHEMA || ''' AS SCHEMA_NM, ''' || TABLE_NAME || ''' AS TABLE_NM, ' ||
        'SUM(CASE WHEN HEADER__OPERATION = ''INSERT'' THEN 1 ELSE 0 END) AS INS_60D, ' ||
        'SUM(CASE WHEN HEADER__OPERATION = ''UPDATE'' THEN 1 ELSE 0 END) AS UPD_60D, ' ||
        'SUM(CASE WHEN HEADER__OPERATION = ''DELETE'' THEN 1 ELSE 0 END) AS DEL_60D, ' ||
        'SUM(CASE WHEN HEADER__OPERATION = ''BEFOREIMAGE'' THEN 1 ELSE 0 END) AS BEFOREIMAGE_60D, ' ||
        'SUM(CASE WHEN HEADER__OPERATION IN (''INSERT'',''UPDATE'',''DELETE'') THEN 1 ELSE 0 END) AS TOT_60D, ' ||
        'COUNT(DISTINCT DATE(HEADER__TIMESTAMP)) AS DAYS_LOADED_60D, ' ||
        'MAX(DATE(HEADER__TIMESTAMP)) AS LAST_LOAD_DT ' ||
        'FROM PROD_LANDING.' || TABLE_SCHEMA || '.' || TABLE_NAME || ' ' ||
        'WHERE HEADER__TIMESTAMP >= DATEADD(DAY, -60, CURRENT_DATE())',
        ' UNION ALL '
    ) WITHIN GROUP (ORDER BY TABLE_NAME)
    INTO :union_sql
    FROM PROD_LANDING.INFORMATION_SCHEMA.TABLES
    WHERE TABLE_SCHEMA = UPPER(:target_schema)
      AND TABLE_TYPE = 'BASE TABLE'
      AND TABLE_NAME LIKE '%\\_\\_CT' ESCAPE '\\';

    -- ------------------------------------------------------------------------
    -- Step B: Wrap with the full distribution analysis logic
    -- ------------------------------------------------------------------------
    full_query := '
        WITH table_totals AS (' || :union_sql || '),

        table_metrics AS (
            SELECT
                SCHEMA_NM, TABLE_NM,
                INS_60D, UPD_60D, DEL_60D, BEFOREIMAGE_60D, TOT_60D,
                DAYS_LOADED_60D, LAST_LOAD_DT,
                DATEDIFF(DAY, LAST_LOAD_DT, CURRENT_DATE()) AS DAYS_SINCE_LAST_LOAD,
                CASE WHEN DAYS_LOADED_60D > 0 THEN TOT_60D / DAYS_LOADED_60D ELSE 0 END AS AVG_RECS_PER_LOAD_DAY,
                CASE WHEN TOT_60D > 0 THEN ROUND(100.0 * INS_60D / TOT_60D, 1) ELSE 0 END AS INSERT_PCT,
                CASE WHEN TOT_60D > 0 THEN ROUND(100.0 * UPD_60D / TOT_60D, 1) ELSE 0 END AS UPDATE_PCT,
                CASE WHEN TOT_60D > 0 THEN ROUND(100.0 * DEL_60D / TOT_60D, 1) ELSE 0 END AS DELETE_PCT,
                CASE
                    WHEN UPD_60D = 0 AND BEFOREIMAGE_60D = 0   THEN ''CDC_OK_NO_UPDATES''
                    WHEN UPD_60D = BEFOREIMAGE_60D             THEN ''CDC_OK''
                    WHEN BEFOREIMAGE_60D = 0 AND UPD_60D > 0   THEN ''CDC_NO_BEFOREIMAGE''
                    ELSE ''CDC_MISMATCH_INVESTIGATE''
                END AS CDC_INTEGRITY_FLAG
            FROM table_totals
        ),

        schema_stats AS (
            SELECT
                AVG(TOT_60D)                                            AS SCHEMA_AVG_TOT,
                STDDEV(TOT_60D)                                         AS SCHEMA_STDDEV_TOT,
                PERCENTILE_CONT(0.10) WITHIN GROUP (ORDER BY TOT_60D)   AS SCHEMA_P10,
                PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY TOT_60D)   AS SCHEMA_Q1,
                PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY TOT_60D)   AS SCHEMA_MEDIAN,
                PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY TOT_60D)   AS SCHEMA_Q3,
                PERCENTILE_CONT(0.90) WITHIN GROUP (ORDER BY TOT_60D)   AS SCHEMA_P90,
                COUNT(*)                                                AS ACTIVE_TABLES,
                (SELECT COUNT(*) FROM table_metrics)                    AS TOTAL_TABLES_IN_SCHEMA,
                (SELECT COUNT(*) FROM table_metrics WHERE TOT_60D = 0)  AS DORMANT_TABLES
            FROM table_metrics
            WHERE TOT_60D > 0
        ),

        ranked_tables AS (
            SELECT
                m.*, s.*,
                RANK() OVER (ORDER BY m.TOT_60D DESC)                          AS ACTIVITY_RANK,
                ROUND(100.0 * PERCENT_RANK() OVER (ORDER BY m.TOT_60D), 1)     AS PERCENTILE_IN_SCHEMA
            FROM table_metrics m
            CROSS JOIN schema_stats s
        )

        SELECT
            SCHEMA_NM,
            TABLE_NM,
            CURRENT_DATE() AS CHECK_DT,
            ACTIVITY_RANK,
            PERCENTILE_IN_SCHEMA,
            LAST_LOAD_DT,
            DAYS_SINCE_LAST_LOAD,

            TOT_60D AS TOTAL_60D,
            INS_60D AS INSERTS_60D,
            UPD_60D AS UPDATES_60D,
            DEL_60D AS DELETES_60D,
            INSERT_PCT,
            UPDATE_PCT,
            DELETE_PCT,
            DAYS_LOADED_60D,
            ROUND(AVG_RECS_PER_LOAD_DAY, 0) AS AVG_RECS_PER_LOAD_DAY,

            BEFOREIMAGE_60D,
            CDC_INTEGRITY_FLAG,

            ROUND(SCHEMA_AVG_TOT, 0)    AS SCHEMA_AVG_60D,
            ROUND(SCHEMA_STDDEV_TOT, 0) AS SCHEMA_STDDEV_60D,
            ROUND(SCHEMA_MEDIAN, 0)     AS SCHEMA_MEDIAN_60D,
            ROUND(SCHEMA_P10, 0)        AS SCHEMA_P10,
            ROUND(SCHEMA_P90, 0)        AS SCHEMA_P90,
            ACTIVE_TABLES,
            DORMANT_TABLES,
            TOTAL_TABLES_IN_SCHEMA,

            CASE
                WHEN TOT_60D = 0           THEN ''PCT_DORMANT''
                WHEN TOT_60D <= SCHEMA_P10 THEN ''PCT_BOTTOM_10''
                WHEN TOT_60D >= SCHEMA_P90 THEN ''PCT_TOP_10''
                ELSE ''PCT_NORMAL''
            END AS ANOMALY_FLAG_PERCENTILE,

            GREATEST(ROUND(SCHEMA_AVG_TOT - 2 * SCHEMA_STDDEV_TOT, 0), 0) AS STDDEV_LOWER_BOUND,
            ROUND(SCHEMA_AVG_TOT + 2 * SCHEMA_STDDEV_TOT, 0)              AS STDDEV_UPPER_BOUND,
            CASE
                WHEN TOT_60D = 0                                                    THEN ''STDDEV_DORMANT''
                WHEN SCHEMA_STDDEV_TOT = 0                                          THEN ''STDDEV_NOT_APPLICABLE''
                WHEN TOT_60D < GREATEST(SCHEMA_AVG_TOT - 2 * SCHEMA_STDDEV_TOT, 0)  THEN ''STDDEV_BELOW_SCHEMA_NORM''
                WHEN TOT_60D > (SCHEMA_AVG_TOT + 2 * SCHEMA_STDDEV_TOT)             THEN ''STDDEV_ABOVE_SCHEMA_NORM''
                ELSE ''STDDEV_OK''
            END AS ANOMALY_FLAG_STDDEV,

            GREATEST(ROUND(SCHEMA_Q1 - 1.5 * (SCHEMA_Q3 - SCHEMA_Q1), 0), 0) AS IQR_LOWER_BOUND,
            ROUND(SCHEMA_Q3 + 1.5 * (SCHEMA_Q3 - SCHEMA_Q1), 0)              AS IQR_UPPER_BOUND,
            CASE
                WHEN TOT_60D = 0                                                            THEN ''IQR_DORMANT''
                WHEN (SCHEMA_Q3 - SCHEMA_Q1) = 0                                            THEN ''IQR_NOT_APPLICABLE''
                WHEN TOT_60D < GREATEST(SCHEMA_Q1 - 1.5 * (SCHEMA_Q3 - SCHEMA_Q1), 0)       THEN ''IQR_OUTLIER_LOW''
                WHEN TOT_60D > (SCHEMA_Q3 + 1.5 * (SCHEMA_Q3 - SCHEMA_Q1))                  THEN ''IQR_OUTLIER_HIGH''
                ELSE ''IQR_OK''
            END AS ANOMALY_FLAG_IQR,

            CASE
                WHEN TOT_60D = 0          THEN ''THRESHOLD_DORMANT''
                WHEN TOT_60D < 600        THEN ''THRESHOLD_VERY_LOW''
                WHEN TOT_60D > 60000000   THEN ''THRESHOLD_VERY_HIGH''
                ELSE ''THRESHOLD_NORMAL''
            END AS ANOMALY_FLAG_THRESHOLD,

            CASE
                WHEN TOT_60D = 0                                            THEN ''DORMANT''
                WHEN TOT_60D > (SCHEMA_AVG_TOT + 2 * SCHEMA_STDDEV_TOT)     THEN ''EXTREME''
                WHEN TOT_60D >= SCHEMA_P90                                  THEN ''HIGH''
                WHEN TOT_60D <= SCHEMA_P10                                  THEN ''LOW''
                ELSE ''NORMAL''
            END AS ACTIVITY_LEVEL,

            CASE
                WHEN TOT_60D = 0                                                                            THEN ''INVESTIGATE_DORMANT''
                WHEN CDC_INTEGRITY_FLAG = ''CDC_MISMATCH_INVESTIGATE''                                      THEN ''INVESTIGATE_CDC''
                WHEN TOT_60D <= SCHEMA_P10
                  OR TOT_60D >= SCHEMA_P90
                  OR (SCHEMA_STDDEV_TOT > 0 AND TOT_60D < GREATEST(SCHEMA_AVG_TOT - 2 * SCHEMA_STDDEV_TOT, 0))
                  OR (SCHEMA_STDDEV_TOT > 0 AND TOT_60D > (SCHEMA_AVG_TOT + 2 * SCHEMA_STDDEV_TOT))
                  OR ((SCHEMA_Q3 - SCHEMA_Q1) > 0 AND TOT_60D < GREATEST(SCHEMA_Q1 - 1.5 * (SCHEMA_Q3 - SCHEMA_Q1), 0))
                  OR ((SCHEMA_Q3 - SCHEMA_Q1) > 0 AND TOT_60D > (SCHEMA_Q3 + 1.5 * (SCHEMA_Q3 - SCHEMA_Q1)))
                  OR TOT_60D < 600
                  OR TOT_60D > 60000000
                    THEN ''INVESTIGATE''
                ELSE ''OK''
            END AS OVERALL_FLAG

        FROM ranked_tables
        ORDER BY
            CASE
                WHEN TOT_60D = 0 THEN 1
                WHEN CDC_INTEGRITY_FLAG = ''CDC_MISMATCH_INVESTIGATE'' THEN 2
                WHEN TOT_60D <= SCHEMA_P10 THEN 3
                WHEN TOT_60D >= SCHEMA_P90 THEN 4
                ELSE 5
            END,
            TOT_60D DESC';

    -- ------------------------------------------------------------------------
    -- Step C: Execute the full query and return results
    -- ------------------------------------------------------------------------
    res := (EXECUTE IMMEDIATE :full_query);
    RETURN TABLE(res);
END;


/* ============================================================================
   USAGE NOTES
   ----------------------------------------------------------------------------
   To analyze another schema:
     - Change the value of target_schema in the DECLARE section above
     - Valid values: 'ING', 'FRATDB', 'RPLUS', 'L70', 'DCLM', 'SAP'
     - Highlight the entire block (DECLARE ... END;) and run

   READ-ONLY GUARANTEE:
     - No CREATE statements
     - No INSERT statements
     - No UPDATE / DELETE statements
     - Only SELECT and INFORMATION_SCHEMA reads
     - Safe to run on PROD as a Data Engineer with SELECT-only access

   PERFORMANCE TIPS:
     - First run: 30-90 seconds (depends on schema size)
     - Same-day reruns: 2-10 seconds (Snowflake result cache)
     - If too slow, temporarily scale warehouse:
         ALTER WAREHOUSE PROD_DATA_ENGINEER_WH SET WAREHOUSE_SIZE = 'MEDIUM';
         (run the block)
         ALTER WAREHOUSE PROD_DATA_ENGINEER_WH SET WAREHOUSE_SIZE = 'X-SMALL';
============================================================================ */
