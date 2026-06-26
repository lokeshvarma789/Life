/* ============================================================================
   STEP 5 - DAILY LOAD VOLUME AND FREQUENCY VALIDATION
   ----------------------------------------------------------------------------
   Validates ONE CT table at a time across all dimensions Sadesh specified:

     1. Today's INSERT / UPDATE / DELETE counts (per HEADER__OPERATION)
     2. Compare against 60-day historical trends
     3. STDDEV deviation flag (mean +/- 2 stddev)
     4. IQR outlier flag (Q1 - 1.5*IQR, Q3 + 1.5*IQR)
     5. Clear differentiation between STDDEV vs IQR methods
     6. Expected frequency detection (DAILY / WEEKLY / MONTHLY / IRREGULAR)
     7. Frequency anomaly flag (missed / delayed intervals)
     8. Overall flag combining all checks

   ARCHITECTURE CONTEXT:
     Database  : PROD_LANDING
     Schemas   : ING, FRATDB, RPLUS (Qlik)  |  L70, DCLM, SAP (Talend)
     CT tables : end with '__CT', contain HEADER__OPERATION, HEADER__TIMESTAMP

   KNOWN CADENCE QUIRKS (do NOT flag as anomalies):
     - L70 tables: Tue-Sat only (no Mon/Sun loads = NORMAL)
     - DC69__CT and similar: may be monthly (verify with team)
     - Some tables in ING: weekly batch (e.g. TZGG2__CT runs Sundays)

   HOW TO USE:
     1. Update the FROM clause + IDENTITY labels (3 places marked with -->)
     2. Run the entire script
     3. Read OVERALL_FLAG first, then drill into method-specific flags

   AUDIT TRAIL:
     Author : Lokesh Varma
     Date   : ____________
     Step   : 5 of monitoring suite
============================================================================ */

USE ROLE SF_PROD_DATA_ENGINEER;
USE WAREHOUSE PROD_DATA_ENGINEER_WH;
USE DATABASE PROD_LANDING;


/* ============================================================================
   MAIN QUERY
============================================================================ */

WITH

-- ----------------------------------------------------------------------------
-- Step 1: Pull 60 days of daily aggregated operations from the target CT table
--         Each row = one day = one (INSERT, UPDATE, DELETE, TOTAL) bucket
-- ----------------------------------------------------------------------------
daily_ops AS (
    SELECT
        DATE(HEADER__TIMESTAMP)                                                  AS LOAD_DT,
        DAYOFWEEK(HEADER__TIMESTAMP)                                             AS DAY_OF_WEEK_NUM,
        DAYNAME(HEADER__TIMESTAMP)                                               AS DAY_OF_WEEK,
        SUM(CASE WHEN HEADER__OPERATION = 'INSERT' THEN 1 ELSE 0 END)            AS INS_CNT,
        SUM(CASE WHEN HEADER__OPERATION = 'UPDATE' THEN 1 ELSE 0 END)            AS UPD_CNT,
        SUM(CASE WHEN HEADER__OPERATION = 'DELETE' THEN 1 ELSE 0 END)            AS DEL_CNT,
        COUNT(*)                                                                 AS TOT_CNT
    FROM PROD_LANDING.ING.TPOL__CT                                              -- > CHANGE TARGET TABLE
    WHERE HEADER__TIMESTAMP >= DATEADD(DAY, -60, CURRENT_DATE())
    GROUP BY DATE(HEADER__TIMESTAMP), DAYOFWEEK(HEADER__TIMESTAMP), DAYNAME(HEADER__TIMESTAMP)
),

-- ----------------------------------------------------------------------------
-- Step 2: Calculate the gap (in days) between consecutive load dates
--         Used to detect the table's natural cadence (daily vs weekly vs monthly)
-- ----------------------------------------------------------------------------
load_intervals AS (
    SELECT
        LOAD_DT,
        DATEDIFF(DAY, LAG(LOAD_DT) OVER (ORDER BY LOAD_DT), LOAD_DT) AS GAP_DAYS
    FROM daily_ops
    WHERE LOAD_DT < CURRENT_DATE()
),

-- ----------------------------------------------------------------------------
-- Step 3: Build historical baseline (last 60 days, excluding today)
--         All statistical inputs for STDDEV + IQR are derived here.
-- ----------------------------------------------------------------------------
hist_stats AS (
    SELECT
        COUNT(*)                                                       AS DAYS_LOADED_60D,

        -- Total volume stats
        AVG(TOT_CNT)                                                   AS HIST_AVG_TOT,
        STDDEV(TOT_CNT)                                                AS HIST_STDDEV_TOT,
        MIN(TOT_CNT)                                                   AS HIST_MIN_TOT,
        MAX(TOT_CNT)                                                   AS HIST_MAX_TOT,
        PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY TOT_CNT)          AS HIST_Q1_TOT,
        PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY TOT_CNT)          AS HIST_MEDIAN_TOT,
        PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY TOT_CNT)          AS HIST_Q3_TOT,

        -- Per-operation averages (Sadesh asked for separate I/U/D analysis)
        AVG(INS_CNT)                                                   AS HIST_AVG_INS,
        STDDEV(INS_CNT)                                                AS HIST_STDDEV_INS,
        AVG(UPD_CNT)                                                   AS HIST_AVG_UPD,
        STDDEV(UPD_CNT)                                                AS HIST_STDDEV_UPD,
        AVG(DEL_CNT)                                                   AS HIST_AVG_DEL,
        STDDEV(DEL_CNT)                                                AS HIST_STDDEV_DEL
    FROM daily_ops
    WHERE LOAD_DT < CURRENT_DATE()
),

-- ----------------------------------------------------------------------------
-- Step 4: Frequency stats (median gap is more robust than avg gap)
-- ----------------------------------------------------------------------------
freq_stats AS (
    SELECT
        AVG(GAP_DAYS)                                                  AS AVG_GAP_DAYS,
        PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY GAP_DAYS)         AS MEDIAN_GAP_DAYS,
        MAX(GAP_DAYS)                                                  AS MAX_GAP_DAYS,
        (SELECT MAX(LOAD_DT) FROM daily_ops)                           AS LAST_LOAD_DT
    FROM load_intervals
    WHERE GAP_DAYS IS NOT NULL
),

-- ----------------------------------------------------------------------------
-- Step 5: Today's actual load (returns 0 if nothing loaded yet)
-- ----------------------------------------------------------------------------
today_ops AS (
    SELECT
        COALESCE(SUM(INS_CNT), 0)                                      AS INSERTS_TODAY,
        COALESCE(SUM(UPD_CNT), 0)                                      AS UPDATES_TODAY,
        COALESCE(SUM(DEL_CNT), 0)                                      AS DELETES_TODAY,
        COALESCE(SUM(TOT_CNT), 0)                                      AS TOTAL_TODAY
    FROM daily_ops
    WHERE LOAD_DT = CURRENT_DATE()
)


/* ============================================================================
   FINAL OUTPUT - ONE ROW PER TABLE WITH ALL CHECKS
============================================================================ */
SELECT
    /* ====================================================================== */
    /* SECTION 1: IDENTITY                                                    */
    /* ====================================================================== */
    'ING'                                                              AS SCHEMA_NM,        -- > CHANGE LABEL
    'TPOL__CT'                                                         AS TABLE_NM,         -- > CHANGE LABEL
    CURRENT_DATE()                                                     AS CHECK_DT,
    DAYNAME(CURRENT_DATE())                                            AS CHECK_DAY,

    -- Cadence classification based on median gap (median is robust against outliers)
    CASE
        WHEN f.MEDIAN_GAP_DAYS IS NULL                                 THEN 'UNKNOWN'
        WHEN f.MEDIAN_GAP_DAYS <= 1.5                                  THEN 'DAILY'
        WHEN f.MEDIAN_GAP_DAYS BETWEEN 1.5 AND 3                       THEN 'BUSINESS_DAILY'   -- skips weekends
        WHEN f.MEDIAN_GAP_DAYS BETWEEN 3 AND 8                         THEN 'WEEKLY'
        WHEN f.MEDIAN_GAP_DAYS BETWEEN 8 AND 35                        THEN 'MONTHLY'
        ELSE 'IRREGULAR'
    END                                                                AS LOAD_CADENCE,

    /* ====================================================================== */
    /* SECTION 2: TODAY'S LOAD (Inserts / Updates / Deletes / Total)          */
    /* ====================================================================== */
    t.INSERTS_TODAY,
    t.UPDATES_TODAY,
    t.DELETES_TODAY,
    t.TOTAL_TODAY,

    /* ====================================================================== */
    /* SECTION 3: 60-DAY HISTORICAL BASELINE                                  */
    /* ====================================================================== */
    h.DAYS_LOADED_60D,
    ROUND(h.HIST_AVG_TOT, 0)                                           AS HIST_AVG_TOTAL,
    ROUND(h.HIST_STDDEV_TOT, 0)                                        AS HIST_STDDEV_TOTAL,
    ROUND(h.HIST_MEDIAN_TOT, 0)                                        AS HIST_MEDIAN_TOTAL,
    h.HIST_MIN_TOT                                                     AS HIST_MIN_TOTAL,
    h.HIST_MAX_TOT                                                     AS HIST_MAX_TOTAL,
    ROUND(h.HIST_Q1_TOT, 0)                                            AS HIST_Q1,
    ROUND(h.HIST_Q3_TOT, 0)                                            AS HIST_Q3,
    ROUND(h.HIST_AVG_INS, 0)                                           AS HIST_AVG_INSERTS,
    ROUND(h.HIST_AVG_UPD, 0)                                           AS HIST_AVG_UPDATES,
    ROUND(h.HIST_AVG_DEL, 0)                                           AS HIST_AVG_DELETES,

    /* ====================================================================== */
    /* SECTION 4: METHOD 1 - STDDEV (mean +/- 2 stddev rule)                  */
    /* ====================================================================== */
    GREATEST(ROUND(h.HIST_AVG_TOT - 2 * h.HIST_STDDEV_TOT, 0), 0)      AS STDDEV_LOWER_BOUND,
    ROUND(h.HIST_AVG_TOT + 2 * h.HIST_STDDEV_TOT, 0)                   AS STDDEV_UPPER_BOUND,
    CASE
        WHEN h.DAYS_LOADED_60D < 5
            THEN 'INSUFFICIENT_HISTORY'
        WHEN h.HIST_STDDEV_TOT = 0
            THEN 'STDDEV_NOT_APPLICABLE'                  -- all days identical, stddev=0
        WHEN t.TOTAL_TODAY < GREATEST(h.HIST_AVG_TOT - 2 * h.HIST_STDDEV_TOT, 0)
            THEN 'STDDEV_BELOW_NORMAL'
        WHEN t.TOTAL_TODAY > (h.HIST_AVG_TOT + 2 * h.HIST_STDDEV_TOT)
            THEN 'STDDEV_ABOVE_NORMAL'
        ELSE 'STDDEV_OK'
    END                                                                AS ANOMALY_FLAG_STDDEV,

    /* ====================================================================== */
    /* SECTION 5: METHOD 2 - IQR (Q1 - 1.5*IQR, Q3 + 1.5*IQR rule)            */
    /* ====================================================================== */
    GREATEST(ROUND(h.HIST_Q1_TOT - 1.5 * (h.HIST_Q3_TOT - h.HIST_Q1_TOT), 0), 0) AS IQR_LOWER_BOUND,
    ROUND(h.HIST_Q3_TOT + 1.5 * (h.HIST_Q3_TOT - h.HIST_Q1_TOT), 0)             AS IQR_UPPER_BOUND,
    CASE
        WHEN h.DAYS_LOADED_60D < 5
            THEN 'INSUFFICIENT_HISTORY'
        WHEN (h.HIST_Q3_TOT - h.HIST_Q1_TOT) = 0
            THEN 'IQR_NOT_APPLICABLE'                     -- all middle 50% identical
        WHEN t.TOTAL_TODAY < GREATEST(h.HIST_Q1_TOT - 1.5 * (h.HIST_Q3_TOT - h.HIST_Q1_TOT), 0)
            THEN 'IQR_OUTLIER_LOW'
        WHEN t.TOTAL_TODAY > (h.HIST_Q3_TOT + 1.5 * (h.HIST_Q3_TOT - h.HIST_Q1_TOT))
            THEN 'IQR_OUTLIER_HIGH'
        ELSE 'IQR_OK'
    END                                                                AS ANOMALY_FLAG_IQR,

    /* ====================================================================== */
    /* SECTION 6: METHOD 3 - FREQUENCY (did load come in on the expected day?)*/
    /* ====================================================================== */
    ROUND(f.MEDIAN_GAP_DAYS, 1)                                        AS EXPECTED_GAP_DAYS,
    f.LAST_LOAD_DT,
    DATEDIFF(DAY, f.LAST_LOAD_DT, CURRENT_DATE())                      AS ACTUAL_DAYS_SINCE_LOAD,
    CASE
        WHEN f.MEDIAN_GAP_DAYS IS NULL
            THEN 'INSUFFICIENT_HISTORY'
        WHEN DATEDIFF(DAY, f.LAST_LOAD_DT, CURRENT_DATE()) > (f.MEDIAN_GAP_DAYS * 2)
            THEN 'FREQUENCY_MISSED_INTERVAL'              -- gap is 2x+ expected
        WHEN DATEDIFF(DAY, f.LAST_LOAD_DT, CURRENT_DATE()) > f.MEDIAN_GAP_DAYS
            THEN 'FREQUENCY_DELAYED'                      -- load is overdue
        ELSE 'FREQUENCY_OK'
    END                                                                AS ANOMALY_FLAG_FREQUENCY,

    /* ====================================================================== */
    /* SECTION 7: PER-OPERATION FLAGS (Sadesh asked for separate I/U/D)       */
    /* ====================================================================== */
    CASE
        WHEN h.DAYS_LOADED_60D < 5 OR h.HIST_STDDEV_INS = 0
            THEN 'INSUFFICIENT_DATA'
        WHEN t.INSERTS_TODAY > (h.HIST_AVG_INS + 2 * h.HIST_STDDEV_INS)
            THEN 'INSERTS_SPIKE'
        WHEN t.INSERTS_TODAY < GREATEST(h.HIST_AVG_INS - 2 * h.HIST_STDDEV_INS, 0)
            THEN 'INSERTS_BELOW_NORMAL'
        ELSE 'INSERTS_OK'
    END                                                                AS FLAG_INSERTS,

    CASE
        WHEN h.DAYS_LOADED_60D < 5 OR h.HIST_STDDEV_UPD = 0
            THEN 'INSUFFICIENT_DATA'
        WHEN t.UPDATES_TODAY > (h.HIST_AVG_UPD + 2 * h.HIST_STDDEV_UPD)
            THEN 'UPDATES_SPIKE'
        WHEN t.UPDATES_TODAY < GREATEST(h.HIST_AVG_UPD - 2 * h.HIST_STDDEV_UPD, 0)
            THEN 'UPDATES_BELOW_NORMAL'
        ELSE 'UPDATES_OK'
    END                                                                AS FLAG_UPDATES,

    CASE
        WHEN h.DAYS_LOADED_60D < 5 OR h.HIST_STDDEV_DEL = 0
            THEN 'INSUFFICIENT_DATA'
        WHEN t.DELETES_TODAY > (h.HIST_AVG_DEL + 2 * h.HIST_STDDEV_DEL)
            THEN 'DELETES_SPIKE'
        WHEN t.DELETES_TODAY < GREATEST(h.HIST_AVG_DEL - 2 * h.HIST_STDDEV_DEL, 0)
            THEN 'DELETES_BELOW_NORMAL'
        ELSE 'DELETES_OK'
    END                                                                AS FLAG_DELETES,

    /* ====================================================================== */
    /* SECTION 8: OVERALL FLAG (triggered if ANY method flagged anything)     */
    /* ====================================================================== */
    CASE
        WHEN h.DAYS_LOADED_60D < 5
            THEN 'REVIEW_NEW_TABLE'
        WHEN t.TOTAL_TODAY < GREATEST(h.HIST_AVG_TOT - 2 * h.HIST_STDDEV_TOT, 0)
          OR t.TOTAL_TODAY > (h.HIST_AVG_TOT + 2 * h.HIST_STDDEV_TOT)
          OR t.TOTAL_TODAY < GREATEST(h.HIST_Q1_TOT - 1.5 * (h.HIST_Q3_TOT - h.HIST_Q1_TOT), 0)
          OR t.TOTAL_TODAY > (h.HIST_Q3_TOT + 1.5 * (h.HIST_Q3_TOT - h.HIST_Q1_TOT))
          OR DATEDIFF(DAY, f.LAST_LOAD_DT, CURRENT_DATE()) > (f.MEDIAN_GAP_DAYS * 2)
            THEN 'INVESTIGATE'
        ELSE 'OK'
    END                                                                AS OVERALL_FLAG

FROM hist_stats h
CROSS JOIN today_ops t
CROSS JOIN freq_stats f;
