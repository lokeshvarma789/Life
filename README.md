/* ============================================================================
   STEP 5 - DAILY LOAD VOLUME AND FREQUENCY VALIDATION
   ----------------------------------------------------------------------------
   Validates ONE CT table at a time against its 60-day historical pattern.

   WHAT IT CHECKS (per Sadesh's requirements):
     - Today's INSERT / UPDATE / DELETE counts
     - STDDEV deviation from historical mean (2 stddev rule)
     - IQR outlier detection (Q1 - 1.5*IQR to Q3 + 1.5*IQR)
     - Load frequency vs expected interval
     - Overall flag combining all three methods

   HOW TO USE:
     1. Change the two SET values below (schema + table)
     2. Run the whole script
     3. Read OVERALL_FLAG first, then drill into individual method flags

   AVAILABLE SCHEMAS : ING, L70, FRATDB, DCLM, DI, LTC, SF, SEI
   EXAMPLE TABLES    : ING.TPOL__CT, L70.BTRLR__CT, FRATDB.MEMBER__CT
============================================================================ */

USE ROLE SF_PROD_DATA_ENGINEER;
USE WAREHOUSE PROD_DATA_ENGINEER_WH;
USE DATABASE PROD_LANDING;

-- (!) CHANGE THESE TWO LINES TO TARGET A DIFFERENT TABLE
SET CHECK_SCHEMA = 'ING';
SET CHECK_TABLE  = 'TPOL__CT';


/* ============================================================================
   MAIN QUERY
   Reads one CT table, builds 60-day daily history, calculates all 3 methods.
============================================================================ */

WITH

-- Step 1: Pull last 60 days of operations from the target CT table
daily_ops AS (
    SELECT
        DATE(HEADER__TIMESTAMP) AS LOAD_DT,
        SUM(CASE WHEN HEADER__OPERATION = 'INSERT' THEN 1 ELSE 0 END) AS INS_CNT,
        SUM(CASE WHEN HEADER__OPERATION = 'UPDATE' THEN 1 ELSE 0 END) AS UPD_CNT,
        SUM(CASE WHEN HEADER__OPERATION = 'DELETE' THEN 1 ELSE 0 END) AS DEL_CNT,
        COUNT(*) AS TOT_CNT
    FROM IDENTIFIER('PROD_LANDING.' || $CHECK_SCHEMA || '.' || $CHECK_TABLE)
    WHERE HEADER__TIMESTAMP >= DATEADD(DAY, -60, CURRENT_DATE())
    GROUP BY DATE(HEADER__TIMESTAMP)
),

-- Step 2: Calculate gaps between consecutive load dates (for frequency check)
load_gaps AS (
    SELECT
        LOAD_DT,
        DATEDIFF(DAY, LAG(LOAD_DT) OVER (ORDER BY LOAD_DT), LOAD_DT) AS GAP_DAYS
    FROM daily_ops
    WHERE LOAD_DT < CURRENT_DATE()
),

-- Step 3: Build historical baseline (past 60 days, excluding today)
hist_stats AS (
    SELECT
        COUNT(*)                                                  AS DAYS_LOADED_60D,
        AVG(TOT_CNT)                                              AS HIST_AVG,
        STDDEV(TOT_CNT)                                           AS HIST_STDDEV,
        MIN(TOT_CNT)                                              AS HIST_MIN,
        MAX(TOT_CNT)                                              AS HIST_MAX,
        PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY TOT_CNT)     AS HIST_Q1,
        PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY TOT_CNT)     AS HIST_MEDIAN,
        PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY TOT_CNT)     AS HIST_Q3,
        AVG(INS_CNT)                                              AS HIST_AVG_INS,
        AVG(UPD_CNT)                                              AS HIST_AVG_UPD,
        AVG(DEL_CNT)                                              AS HIST_AVG_DEL,
        (SELECT AVG(GAP_DAYS) FROM load_gaps WHERE GAP_DAYS IS NOT NULL) AS AVG_GAP_DAYS,
        (SELECT MAX(LOAD_DT)  FROM daily_ops)                     AS LAST_LOAD_DT
    FROM daily_ops
    WHERE LOAD_DT < CURRENT_DATE()
),

-- Step 4: Today's load (returns 0 if nothing loaded yet)
today_ops AS (
    SELECT
        COALESCE(SUM(INS_CNT), 0) AS INSERTS_TODAY,
        COALESCE(SUM(UPD_CNT), 0) AS UPDATES_TODAY,
        COALESCE(SUM(DEL_CNT), 0) AS DELETES_TODAY,
        COALESCE(SUM(TOT_CNT), 0) AS TOTAL_TODAY
    FROM daily_ops
    WHERE LOAD_DT = CURRENT_DATE()
)


/* ============================================================================
   FINAL OUTPUT - one row showing all checks
============================================================================ */
SELECT
    -- ===== IDENTITY =====
    $CHECK_SCHEMA                                                 AS SCHEMA_NM,
    $CHECK_TABLE                                                  AS TABLE_NM,
    CURRENT_DATE()                                                AS CHECK_DT,

    CASE
        WHEN h.AVG_GAP_DAYS IS NULL    THEN 'UNKNOWN'
        WHEN h.AVG_GAP_DAYS <= 1.5     THEN 'DAILY'
        WHEN h.AVG_GAP_DAYS <= 8       THEN 'WEEKLY'
        WHEN h.AVG_GAP_DAYS <= 35      THEN 'MONTHLY'
        ELSE 'IRREGULAR'
    END                                                           AS LOAD_CADENCE,

    -- ===== TODAY'S LOAD =====
    t.INSERTS_TODAY,
    t.UPDATES_TODAY,
    t.DELETES_TODAY,
    t.TOTAL_TODAY,

    -- ===== HISTORY (60 days) =====
    h.DAYS_LOADED_60D,
    ROUND(h.HIST_AVG, 0)                                          AS HIST_AVG_60D,
    ROUND(h.HIST_STDDEV, 0)                                       AS HIST_STDDEV_60D,
    ROUND(h.HIST_AVG_INS, 0)                                      AS HIST_AVG_INSERTS,
    ROUND(h.HIST_AVG_UPD, 0)                                      AS HIST_AVG_UPDATES,
    ROUND(h.HIST_AVG_DEL, 0)                                      AS HIST_AVG_DELETES,
    h.HIST_MIN                                                    AS HIST_MIN_60D,
    h.HIST_MAX                                                    AS HIST_MAX_60D,
    ROUND(h.HIST_Q1, 0)                                           AS HIST_Q1,
    ROUND(h.HIST_MEDIAN, 0)                                       AS HIST_MEDIAN,
    ROUND(h.HIST_Q3, 0)                                           AS HIST_Q3,

    -- ===== METHOD 1: STDDEV (2 standard deviations from mean) =====
    GREATEST(ROUND(h.HIST_AVG - 2 * h.HIST_STDDEV, 0), 0)         AS STDDEV_LOWER_BOUND,
    ROUND(h.HIST_AVG + 2 * h.HIST_STDDEV, 0)                      AS STDDEV_UPPER_BOUND,
    CASE
        WHEN h.DAYS_LOADED_60D < 5
            THEN 'INSUFFICIENT_HISTORY'
        WHEN t.TOTAL_TODAY < GREATEST(h.HIST_AVG - 2 * h.HIST_STDDEV, 0)
            THEN 'STDDEV_BELOW_NORMAL'
        WHEN t.TOTAL_TODAY > (h.HIST_AVG + 2 * h.HIST_STDDEV)
            THEN 'STDDEV_ABOVE_NORMAL'
        ELSE 'STDDEV_OK'
    END                                                           AS ANOMALY_FLAG_STDDEV,

    -- ===== METHOD 2: IQR (Q1 - 1.5*IQR to Q3 + 1.5*IQR) =====
    GREATEST(ROUND(h.HIST_Q1 - 1.5 * (h.HIST_Q3 - h.HIST_Q1), 0), 0) AS IQR_LOWER_BOUND,
    ROUND(h.HIST_Q3 + 1.5 * (h.HIST_Q3 - h.HIST_Q1), 0)             AS IQR_UPPER_BOUND,
    CASE
        WHEN h.DAYS_LOADED_60D < 5
            THEN 'INSUFFICIENT_HISTORY'
        WHEN t.TOTAL_TODAY < GREATEST(h.HIST_Q1 - 1.5 * (h.HIST_Q3 - h.HIST_Q1), 0)
            THEN 'IQR_OUTLIER_LOW'
        WHEN t.TOTAL_TODAY > (h.HIST_Q3 + 1.5 * (h.HIST_Q3 - h.HIST_Q1))
            THEN 'IQR_OUTLIER_HIGH'
        ELSE 'IQR_OK'
    END                                                           AS ANOMALY_FLAG_IQR,

    -- ===== METHOD 3: FREQUENCY (did the load come in on time?) =====
    ROUND(h.AVG_GAP_DAYS, 1)                                      AS EXPECTED_GAP_DAYS,
    h.LAST_LOAD_DT,
    DATEDIFF(DAY, h.LAST_LOAD_DT, CURRENT_DATE())                 AS ACTUAL_DAYS_SINCE_LOAD,
    CASE
        WHEN h.AVG_GAP_DAYS IS NULL
            THEN 'INSUFFICIENT_HISTORY'
        WHEN DATEDIFF(DAY, h.LAST_LOAD_DT, CURRENT_DATE()) > (h.AVG_GAP_DAYS * 2)
            THEN 'FREQUENCY_MISSED_INTERVAL'
        WHEN DATEDIFF(DAY, h.LAST_LOAD_DT, CURRENT_DATE()) > h.AVG_GAP_DAYS
            THEN 'FREQUENCY_DELAYED'
        ELSE 'FREQUENCY_OK'
    END                                                           AS ANOMALY_FLAG_FREQUENCY,

    -- ===== OVERALL: triggered if ANY method flagged something =====
    CASE
        WHEN h.DAYS_LOADED_60D < 5
            THEN 'REVIEW_NEW_TABLE'
        WHEN t.TOTAL_TODAY < GREATEST(h.HIST_AVG - 2 * h.HIST_STDDEV, 0)
          OR t.TOTAL_TODAY > (h.HIST_AVG + 2 * h.HIST_STDDEV)
          OR t.TOTAL_TODAY < GREATEST(h.HIST_Q1 - 1.5 * (h.HIST_Q3 - h.HIST_Q1), 0)
          OR t.TOTAL_TODAY > (h.HIST_Q3 + 1.5 * (h.HIST_Q3 - h.HIST_Q1))
          OR DATEDIFF(DAY, h.LAST_LOAD_DT, CURRENT_DATE()) > (h.AVG_GAP_DAYS * 2)
            THEN 'INVESTIGATE'
        ELSE 'OK'
    END                                                           AS OVERALL_FLAG

FROM hist_stats h
CROSS JOIN today_ops t;
