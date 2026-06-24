
/* ============================================================================
   STEP 5 - DAILY LOAD VOLUME AND FREQUENCY VALIDATION
   ----------------------------------------------------------------------------
   For every CT table: check today's INSERT/UPDATE/DELETE counts against
   60-day history using THREE methods (STDDEV, IQR, Frequency) and flag
   any table where today's load looks abnormal.

   HOW TO READ THE OUTPUT (left to right):
     IDENTITY        : SCHEMA, TABLE, CHECK_DATE, LOAD_CADENCE
     TODAY'S LOAD    : INSERTS_TODAY, UPDATES_TODAY, DELETES_TODAY, TOTAL_TODAY
     HISTORY 60D     : AVG / STDDEV / Q1 / MEDIAN / Q3 / MIN / MAX / DAYS_LOADED
     METHOD 1 STDDEV : bounds + ANOMALY_FLAG_STDDEV (2-stddev rule)
     METHOD 2 IQR    : bounds + ANOMALY_FLAG_IQR    (1.5 x IQR rule)
     METHOD 3 FREQ   : ANOMALY_FLAG_FREQUENCY       (missed expected interval)
     FINAL           : OVERALL_FLAG (set if ANY method flagged it)

   HOW TO USE :
     1. Set CHECK_SCHEMA below (ING, L70, FRATDB, DCLM, DI, LTC, SF, SEI)
     2. Run BLOCK 1 to generate UNION ALL across tables
     3. Paste output inside daily_ops CTE in BLOCK 2
     4. Run BLOCK 2 and review OVERALL_FLAG column first
============================================================================ */

USE ROLE SF_PROD_DATA_ENGINEER;
USE WAREHOUSE PROD_DATA_ENGINEER_WH;
USE DATABASE PROD_LANDING;

SET CHECK_SCHEMA = 'ING';


/* ============================================================================
   BLOCK 1 : Generate one row per (table, day) with I/U/D counts
============================================================================ */

SELECT LISTAGG(
    'SELECT ''' || TABLE_SCHEMA || ''' AS SCHEMA_NM, ''' || TABLE_NAME || ''' AS TABLE_NM, ' ||
    'DATE(HEADER__TIMESTAMP) AS LOAD_DT, ' ||
    'SUM(CASE WHEN HEADER__OPERATION = ''INSERT'' THEN 1 ELSE 0 END) AS INS_CNT, ' ||
    'SUM(CASE WHEN HEADER__OPERATION = ''UPDATE'' THEN 1 ELSE 0 END) AS UPD_CNT, ' ||
    'SUM(CASE WHEN HEADER__OPERATION = ''DELETE'' THEN 1 ELSE 0 END) AS DEL_CNT, ' ||
    'COUNT(*) AS TOT_CNT ' ||
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
   BLOCK 2 : MAIN STEP 5 QUERY (paste BLOCK 1 output inside daily_ops CTE)
============================================================================ */

WITH daily_ops AS (
    -- (!) PASTE BLOCK 1 OUTPUT HERE (replace these placeholder lines)
),

-- 60-day baseline per table (excluding today)
hist_stats AS (
    SELECT
        SCHEMA_NM,
        TABLE_NM,
        COUNT(*)                                       AS DAYS_LOADED_60D,
        AVG(TOT_CNT)                                   AS HIST_AVG,
        STDDEV(TOT_CNT)                                AS HIST_STDDEV,
        MIN(TOT_CNT)                                   AS HIST_MIN,
        MAX(TOT_CNT)                                   AS HIST_MAX,
        PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY TOT_CNT) AS HIST_Q1,
        PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY TOT_CNT) AS HIST_MEDIAN,
        PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY TOT_CNT) AS HIST_Q3,
        AVG(INS_CNT)                                   AS HIST_AVG_INS,
        AVG(UPD_CNT)                                   AS HIST_AVG_UPD,
        AVG(DEL_CNT)                                   AS HIST_AVG_DEL,
        AVG(DATEDIFF(DAY, LAG(LOAD_DT) OVER (PARTITION BY SCHEMA_NM, TABLE_NM ORDER BY LOAD_DT), LOAD_DT)) AS AVG_GAP_DAYS
    FROM daily_ops
    WHERE LOAD_DT < CURRENT_DATE()
    GROUP BY SCHEMA_NM, TABLE_NM
),

-- Today's actual load per table
today_ops AS (
    SELECT
        SCHEMA_NM,
        TABLE_NM,
        SUM(INS_CNT) AS INSERTS_TODAY,
        SUM(UPD_CNT) AS UPDATES_TODAY,
        SUM(DEL_CNT) AS DELETES_TODAY,
        SUM(TOT_CNT) AS TOTAL_TODAY
    FROM daily_ops
    WHERE LOAD_DT = CURRENT_DATE()
    GROUP BY SCHEMA_NM, TABLE_NM
),

-- Days since last load (for frequency check)
last_load AS (
    SELECT
        SCHEMA_NM,
        TABLE_NM,
        MAX(LOAD_DT)                              AS LAST_LOAD_DT,
        DATEDIFF(DAY, MAX(LOAD_DT), CURRENT_DATE()) AS DAYS_SINCE_LOAD
    FROM daily_ops
    GROUP BY SCHEMA_NM, TABLE_NM
)

SELECT
    -- IDENTITY
    h.SCHEMA_NM,
    h.TABLE_NM,
    CURRENT_DATE()                                              AS CHECK_DT,
    CASE
        WHEN h.AVG_GAP_DAYS <= 1.5 THEN 'DAILY'
        WHEN h.AVG_GAP_DAYS <= 8   THEN 'WEEKLY'
        WHEN h.AVG_GAP_DAYS <= 35  THEN 'MONTHLY'
        ELSE 'IRREGULAR'
    END                                                         AS LOAD_CADENCE,

    -- TODAY'S LOAD
    COALESCE(t.INSERTS_TODAY, 0)                                AS INSERTS_TODAY,
    COALESCE(t.UPDATES_TODAY, 0)                                AS UPDATES_TODAY,
    COALESCE(t.DELETES_TODAY, 0)                                AS DELETES_TODAY,
    COALESCE(t.TOTAL_TODAY, 0)                                  AS TOTAL_TODAY,

    -- HISTORY 60D
    ROUND(h.HIST_AVG, 0)                                        AS HIST_AVG_60D,
    ROUND(h.HIST_STDDEV, 0)                                     AS HIST_STDDEV_60D,
    ROUND(h.HIST_Q1, 0)                                         AS HIST_Q1_60D,
    ROUND(h.HIST_MEDIAN, 0)                                     AS HIST_MEDIAN_60D,
    ROUND(h.HIST_Q3, 0)                                         AS HIST_Q3_60D,
    h.HIST_MIN                                                  AS HIST_MIN_60D,
    h.HIST_MAX                                                  AS HIST_MAX_60D,
    h.DAYS_LOADED_60D,

    -- METHOD 1: STDDEV (2 standard deviations from mean, floor at 0)
    GREATEST(ROUND(h.HIST_AVG - (2 * h.HIST_STDDEV), 0), 0)     AS STDDEV_LOWER_BOUND,
    ROUND(h.HIST_AVG + (2 * h.HIST_STDDEV), 0)                  AS STDDEV_UPPER_BOUND,
    CASE
        WHEN h.HIST_AVG IS NULL OR h.DAYS_LOADED_60D < 5         THEN 'INSUFFICIENT_HISTORY'
        WHEN COALESCE(t.TOTAL_TODAY,0) < GREATEST(h.HIST_AVG - 2*h.HIST_STDDEV, 0)  THEN 'STDDEV_BELOW_NORMAL'
        WHEN COALESCE(t.TOTAL_TODAY,0) > (h.HIST_AVG + 2*h.HIST_STDDEV)             THEN 'STDDEV_ABOVE_NORMAL'
        ELSE 'STDDEV_OK'
    END                                                         AS ANOMALY_FLAG_STDDEV,

    -- METHOD 2: IQR (Q1 - 1.5*IQR  to  Q3 + 1.5*IQR), floor at 0
    GREATEST(ROUND(h.HIST_Q1 - 1.5 * (h.HIST_Q3 - h.HIST_Q1), 0), 0) AS IQR_LOWER_BOUND,
    ROUND(h.HIST_Q3 + 1.5 * (h.HIST_Q3 - h.HIST_Q1), 0)             AS IQR_UPPER_BOUND,
    CASE
        WHEN h.HIST_Q1 IS NULL OR h.DAYS_LOADED_60D < 5 THEN 'INSUFFICIENT_HISTORY'
        WHEN COALESCE(t.TOTAL_TODAY,0) < GREATEST(h.HIST_Q1 - 1.5*(h.HIST_Q3 - h.HIST_Q1), 0) THEN 'IQR_OUTLIER_LOW'
        WHEN COALESCE(t.TOTAL_TODAY,0) > (h.HIST_Q3 + 1.5*(h.HIST_Q3 - h.HIST_Q1))            THEN 'IQR_OUTLIER_HIGH'
        ELSE 'IQR_OK'
    END                                                         AS ANOMALY_FLAG_IQR,

    -- METHOD 3: FREQUENCY (did load come in when expected?)
    ROUND(h.AVG_GAP_DAYS, 1)                                    AS EXPECTED_GAP_DAYS,
    l.DAYS_SINCE_LOAD                                           AS ACTUAL_DAYS_SINCE_LOAD,
    CASE
        WHEN h.AVG_GAP_DAYS IS NULL                                 THEN 'INSUFFICIENT_HISTORY'
        WHEN l.DAYS_SINCE_LOAD > (h.AVG_GAP_DAYS * 2) THEN 'FREQUENCY_MISSED_INTERVAL'
        WHEN l.DAYS_SINCE_LOAD > h.AVG_GAP_DAYS                     THEN 'FREQUENCY_DELAYED'
        ELSE 'FREQUENCY_OK'
    END                                                         AS ANOMALY_FLAG_FREQUENCY,

    -- OVERALL: investigate if ANY method flagged it
    CASE
        WHEN h.DAYS_LOADED_60D < 5 THEN 'REVIEW_NEW_TABLE'
        WHEN (COALESCE(t.TOTAL_TODAY,0) < GREATEST(h.HIST_AVG - 2*h.HIST_STDDEV, 0)
              OR COALESCE(t.TOTAL_TODAY,0) > (h.HIST_AVG + 2*h.HIST_STDDEV)
              OR COALESCE(t.TOTAL_TODAY,0) < GREATEST(h.HIST_Q1 - 1.5*(h.HIST_Q3 - h.HIST_Q1), 0)
              OR COALESCE(t.TOTAL_TODAY,0) > (h.HIST_Q3 + 1.5*(h.HIST_Q3 - h.HIST_Q1))
              OR l.DAYS_SINCE_LOAD > (h.AVG_GAP_DAYS * 2))
            THEN 'INVESTIGATE'
        ELSE 'OK'
    END                                                         AS OVERALL_FLAG

FROM hist_stats h
LEFT JOIN today_ops t ON h.SCHEMA_NM = t.SCHEMA_NM AND h.TABLE_NM = t.TABLE_NM
LEFT JOIN last_load l ON h.SCHEMA_NM = l.SCHEMA_NM AND h.TABLE_NM = l.TABLE_NM
ORDER BY
    CASE WHEN OVERALL_FLAG = 'INVESTIGATE' THEN 1
         WHEN OVERALL_FLAG = 'REVIEW_NEW_TABLE' THEN 2
         ELSE 3 END,
    h.TABLE_NM;
