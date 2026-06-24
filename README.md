
/* ============================================================================
   STEP 5 - DAILY LOAD VOLUME AND FREQUENCY VALIDATION
   ----------------------------------------------------------------------------
   PURPOSE : For every CT table in the schema, compare TODAY's load count
             against its own 60-day historical baseline and flag anomalies.

   PRIMARY METHOD : Standard Deviation (as suggested by Sadesh)
   BACKUP METHOD  : Simple Percentage (for cross-validation)

   HOW TO USE :
     1. Change CHECK_SCHEMA value below.
     2. Run the query.
     3. Review the ANOMALY_FLAG_STDDEV column for INVESTIGATE rows.

   AVAILABLE SCHEMAS : ING, L70, FRATDB, DCLM, DI, LTC, SF, SEI
============================================================================ */

USE ROLE SF_PROD_DATA_ENGINEER;
USE WAREHOUSE PROD_DATA_ENGINEER_WH;
USE DATABASE PROD_LANDING;

SET CHECK_SCHEMA = 'ING';   -- (!) Change schema here


/* ----------------------------------------------------------------------------
   Auto-generate one UNION ALL across every CT table in the schema.
   Run the output of this block, then paste-and-run as the final query.
---------------------------------------------------------------------------- */

SELECT LISTAGG(
    'SELECT ''' || TABLE_SCHEMA || ''' AS SCHEMA_NM, ''' || TABLE_NAME || ''' AS TABLE_NM, ' ||
    'DATE(HEADER__TIMESTAMP) AS LOAD_DT, COUNT(*) AS REC_CNT ' ||
    'FROM PROD_LANDING.' || TABLE_SCHEMA || '.' || TABLE_NAME || ' ' ||
    'WHERE HEADER__TIMESTAMP >= DATEADD(DAY, -60, CURRENT_DATE()) ' ||
    'GROUP BY DATE(HEADER__TIMESTAMP)',
    ' UNION ALL '
) WITHIN GROUP (ORDER BY TABLE_NAME) AS DYNAMIC_SQL
FROM PROD_LANDING.INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = $CHECK_SCHEMA
  AND TABLE_TYPE = 'BASE TABLE'
  AND TABLE_NAME LIKE '%\_\_CT' ESCAPE '\';


/* ============================================================================
   MAIN STEP 5 QUERY
   ----------------------------------------------------------------------------
   Paste the generated UNION ALL from above into the daily_loads CTE below.
============================================================================ */

WITH daily_loads AS (
    -- (!) PASTE the dynamic SQL output here
    SELECT 'ING' AS SCHEMA_NM, 'TPOL__CT' AS TABLE_NM,
           DATE(HEADER__TIMESTAMP) AS LOAD_DT, COUNT(*) AS REC_CNT
    FROM PROD_LANDING.ING.TPOL__CT
    WHERE HEADER__TIMESTAMP >= DATEADD(DAY, -60, CURRENT_DATE())
    GROUP BY DATE(HEADER__TIMESTAMP)
),

-- Per-table baseline: average and stddev from the past 60 days (excluding today)
baseline_stats AS (
    SELECT
        SCHEMA_NM,
        TABLE_NM,
        AVG(REC_CNT)         AS BASELINE_AVG_60D,
        STDDEV(REC_CNT)      AS BASELINE_STDDEV_60D,
        MIN(REC_CNT)         AS BASELINE_MIN_60D,
        MAX(REC_CNT)         AS BASELINE_MAX_60D,
        COUNT(*)             AS HISTORY_DAYS
    FROM daily_loads
    WHERE LOAD_DT < CURRENT_DATE()
    GROUP BY SCHEMA_NM, TABLE_NM
),

-- Today's load count per table
today_load AS (
    SELECT
        SCHEMA_NM,
        TABLE_NM,
        SUM(REC_CNT) AS TODAY_REC_CNT
    FROM daily_loads
    WHERE LOAD_DT = CURRENT_DATE()
    GROUP BY SCHEMA_NM, TABLE_NM
)

-- Final output: each table with both anomaly flags
SELECT
    b.SCHEMA_NM,
    b.TABLE_NM,
    CURRENT_DATE()                                              AS CHECK_DT,
    COALESCE(t.TODAY_REC_CNT, 0)                                AS TODAY_REC_CNT,
    ROUND(b.BASELINE_AVG_60D, 0)                                AS BASELINE_AVG_60D,
    ROUND(b.BASELINE_STDDEV_60D, 0)                             AS BASELINE_STDDEV_60D,
    b.BASELINE_MIN_60D,
    b.BASELINE_MAX_60D,
    b.HISTORY_DAYS,

    -- STDDEV bounds (mean +/- 2 stddev covers ~95% of normal variation)
    ROUND(b.BASELINE_AVG_60D - (2 * b.BASELINE_STDDEV_60D), 0)  AS LOWER_BOUND_STDDEV,
    ROUND(b.BASELINE_AVG_60D + (2 * b.BASELINE_STDDEV_60D), 0)  AS UPPER_BOUND_STDDEV,

    -- PRIMARY FLAG: Standard Deviation method
    CASE
        WHEN b.BASELINE_AVG_60D IS NULL                                    THEN 'NO_HISTORY'
        WHEN COALESCE(t.TODAY_REC_CNT, 0) = 0                              THEN 'INVESTIGATE_ZERO_LOAD'
        WHEN t.TODAY_REC_CNT < (b.BASELINE_AVG_60D - 2 * b.BASELINE_STDDEV_60D) THEN 'INVESTIGATE_BELOW_NORMAL'
        WHEN t.TODAY_REC_CNT > (b.BASELINE_AVG_60D + 2 * b.BASELINE_STDDEV_60D) THEN 'INVESTIGATE_ABOVE_NORMAL'
        ELSE 'OK'
    END                                                         AS ANOMALY_FLAG_STDDEV,

    -- Backup: % of normal for cross-check
    ROUND((COALESCE(t.TODAY_REC_CNT, 0) / NULLIF(b.BASELINE_AVG_60D, 0)) * 100, 1)
                                                                AS PCT_OF_NORMAL,

    -- SECONDARY FLAG: Simple Percentage method (50% / 200% thresholds)
    CASE
        WHEN b.BASELINE_AVG_60D IS NULL OR b.BASELINE_AVG_60D = 0           THEN 'NO_HISTORY'
        WHEN COALESCE(t.TODAY_REC_CNT, 0) = 0                               THEN 'INVESTIGATE_ZERO_LOAD'
        WHEN t.TODAY_REC_CNT < (b.BASELINE_AVG_60D * 0.5)                   THEN 'INVESTIGATE_BELOW_NORMAL'
        WHEN t.TODAY_REC_CNT > (b.BASELINE_AVG_60D * 2.0)                   THEN 'INVESTIGATE_ABOVE_NORMAL'
        ELSE 'OK'
    END                                                         AS ANOMALY_FLAG_PCT

FROM baseline_stats b
LEFT JOIN today_load t
    ON b.SCHEMA_NM = t.SCHEMA_NM
   AND b.TABLE_NM  = t.TABLE_NM
ORDER BY
    CASE WHEN ANOMALY_FLAG_STDDEV LIKE 'INVESTIGATE%' THEN 1 ELSE 2 END,
    b.TABLE_NM;
