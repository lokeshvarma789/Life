
/* ============================================================================
   STEP 8 - SINGLE TABLE VALIDATION  (TPLRT__CT)
   ----------------------------------------------------------------------------
   Purpose: verify the Step 8 logic against one table BEFORE building it
            for all 139 tables in the schema.

   Runs three checks:
     Query 1 - See the actual load history (eyeball the pattern)
     Query 2 - See the gaps between loads (validate the rhythm)
     Query 3 - Apply the Step 8 classification and see the result

   If the classification in Query 3 matches what Queries 1 and 2 tell us
   about the table's real behavior, the logic is sound and we scale it.

   AUTHOR: Lokesh Varma
============================================================================ */


/* ---- Session setup ---- */
USE ROLE SF_PROD_DATA_ENGINEER;
USE WAREHOUSE PROD_DATA_ENGINEER_WH;
USE DATABASE PROD_LANDING;


/* ============================================================================
   QUERY 1 - Load history (eyeball)
   ----------------------------------------------------------------------------
   Shows every distinct load day in the past 180 days with operation counts.
   Look at the pattern: are loads roughly daily? Weekly? Random?
============================================================================ */

SELECT
    DATE(HEADER__TIMESTAMP)                                            AS LOAD_DT,
    COUNT(CASE WHEN HEADER__OPERATION = 'INSERT' THEN 1 END)           AS INSERTS,
    COUNT(CASE WHEN HEADER__OPERATION = 'UPDATE' THEN 1 END)           AS UPDATES,
    COUNT(CASE WHEN HEADER__OPERATION = 'DELETE' THEN 1 END)           AS DELETES,
    COUNT(*)                                                           AS TOTAL_ROWS
FROM PROD_LANDING.ING.TPLRT__CT
WHERE HEADER__TIMESTAMP >= DATEADD(DAY, -180, CURRENT_DATE())
  AND UPPER(NVL(HEADER__OPERATION, '')) <> 'BEFOREIMAGE'
GROUP BY 1
ORDER BY 1 DESC
;


/* ============================================================================
   QUERY 2 - Gaps between loads (validate rhythm)
   ----------------------------------------------------------------------------
   Shows each load day plus the gap in days since the previous load.
   The AVG and MEDIAN of GAP_DAYS is the table's "natural rhythm".
   Our Step 8 logic estimates this as (180 / DAYS_LOADED_180D).
   These two numbers should be close - that's the validation.
============================================================================ */

WITH load_dates AS (
    SELECT DISTINCT DATE(HEADER__TIMESTAMP)                            AS LOAD_DT
    FROM PROD_LANDING.ING.TPLRT__CT
    WHERE HEADER__TIMESTAMP >= DATEADD(DAY, -180, CURRENT_DATE())
      AND UPPER(NVL(HEADER__OPERATION, '')) <> 'BEFOREIMAGE'
),
with_gaps AS (
    SELECT
        LOAD_DT,
        LAG(LOAD_DT) OVER (ORDER BY LOAD_DT)                           AS PREV_LOAD_DT,
        DATEDIFF(DAY, LAG(LOAD_DT) OVER (ORDER BY LOAD_DT), LOAD_DT)   AS GAP_DAYS
    FROM load_dates
)
SELECT
    LOAD_DT,
    PREV_LOAD_DT,
    GAP_DAYS,
    -- Summary stats appear on every row for quick reading
    (SELECT COUNT(*) FROM load_dates)                                  AS TOTAL_LOAD_DAYS_180D,
    (SELECT ROUND(AVG(GAP_DAYS), 2) FROM with_gaps
        WHERE GAP_DAYS IS NOT NULL)                                    AS ACTUAL_AVG_GAP,
    (SELECT MEDIAN(GAP_DAYS) FROM with_gaps
        WHERE GAP_DAYS IS NOT NULL)                                    AS ACTUAL_MEDIAN_GAP,
    (SELECT MAX(GAP_DAYS) FROM with_gaps
        WHERE GAP_DAYS IS NOT NULL)                                    AS ACTUAL_MAX_GAP
FROM with_gaps
ORDER BY LOAD_DT DESC
;


/* ============================================================================
   QUERY 3 - Step 8 classification (compare against Queries 1 and 2)
   ----------------------------------------------------------------------------
   Applies the full Step 8 logic to this one table.
   Compare the result with what Queries 1 and 2 showed:
     - Does DETECTED_CADENCE match the actual pattern?
     - Is AVG_GAP_DAYS close to ACTUAL_AVG_GAP from Query 2?
     - Is FINAL_ACTION appropriate?
============================================================================ */

WITH per_table AS (
    SELECT
        'ING'                                                          AS SCHEMA_NM,
        'TPLRT__CT'                                                    AS TABLE_NM,
        MAX(CASE WHEN UPPER(NVL(HEADER__OPERATION, '')) <> 'BEFOREIMAGE'
                 THEN DATE(HEADER__TIMESTAMP) END)                     AS LAST_LOAD_DT,
        COUNT(DISTINCT CASE WHEN UPPER(NVL(HEADER__OPERATION, '')) <> 'BEFOREIMAGE'
                             AND HEADER__TIMESTAMP >= DATEADD(DAY, -60, CURRENT_DATE())
                             THEN DATE(HEADER__TIMESTAMP) END)         AS DAYS_LOADED_60D,
        COUNT(DISTINCT CASE WHEN UPPER(NVL(HEADER__OPERATION, '')) <> 'BEFOREIMAGE'
                             THEN DATE(HEADER__TIMESTAMP) END)         AS DAYS_LOADED_180D,
        COUNT(CASE WHEN DATE(HEADER__TIMESTAMP) = CURRENT_DATE()
                   AND UPPER(NVL(HEADER__OPERATION, '')) <> 'BEFOREIMAGE'
                   THEN 1 END)                                         AS ROWS_TODAY
    FROM PROD_LANDING.ING.TPLRT__CT
    WHERE HEADER__TIMESTAMP >= DATEADD(DAY, -180, CURRENT_DATE())
),
enriched AS (
    SELECT
        SCHEMA_NM,
        TABLE_NM,
        LAST_LOAD_DT,
        DAYS_LOADED_60D,
        DAYS_LOADED_180D,
        ROWS_TODAY,
        DATEDIFF(DAY, LAST_LOAD_DT, CURRENT_DATE())                    AS DAYS_STALE,
        CASE WHEN DAYS_LOADED_180D > 0
             THEN ROUND(180.0 / DAYS_LOADED_180D, 1) END               AS AVG_GAP_DAYS,
        CASE WHEN DAYS_LOADED_180D > 0
             THEN GREATEST(ROUND(180.0 / DAYS_LOADED_180D * 2.5, 0), 3)
             END                                                       AS ALERT_THRESHOLD_DAYS,
        CASE
            WHEN DAYS_LOADED_180D = 0                THEN 'DORMANT'
            WHEN DAYS_LOADED_180D >= 150             THEN 'DAILY'
            WHEN DAYS_LOADED_180D BETWEEN 40 AND 149 THEN 'MULTI_WEEKLY'
            WHEN DAYS_LOADED_180D BETWEEN 15 AND 39  THEN 'WEEKLY'
            WHEN DAYS_LOADED_180D BETWEEN 5  AND 14  THEN 'BI_MONTHLY'
            WHEN DAYS_LOADED_180D BETWEEN 2  AND 4   THEN 'MONTHLY'
            ELSE                                          'RARE'
        END                                                            AS DETECTED_CADENCE
    FROM per_table
)
SELECT
    CURRENT_DATE()                     AS CHECK_DATE,
    SCHEMA_NM,
    TABLE_NM,
    ROWS_TODAY,
    LAST_LOAD_DT,
    DAYS_STALE,
    DETECTED_CADENCE,
    DAYS_LOADED_60D,
    DAYS_LOADED_180D,
    AVG_GAP_DAYS,
    ALERT_THRESHOLD_DAYS,
    CASE
        WHEN ROWS_TODAY > 0                              THEN 'LOADED_TODAY'
        WHEN DAYS_LOADED_180D = 0                        THEN 'DORMANT'
        WHEN DAYS_LOADED_180D < 3                        THEN 'INSUFFICIENT_HISTORY'
        WHEN DAYS_STALE <= ALERT_THRESHOLD_DAYS          THEN 'EXPECTED_GAP'
        WHEN DAYS_LOADED_180D >= 20                      THEN 'STOPPED_LOADING'
        ELSE                                                  'UNEXPECTED_MISS'
    END                                                                AS MISS_CLASSIFICATION,
    CASE
        WHEN ROWS_TODAY > 0                              THEN 'OK_LOADED_TODAY'
        WHEN DAYS_LOADED_180D = 0                        THEN 'DORMANT_CONFIRM'
        WHEN DAYS_LOADED_180D < 3                        THEN 'INSUFFICIENT_HISTORY'
        WHEN DAYS_STALE <= ALERT_THRESHOLD_DAYS          THEN 'OK'
        WHEN DAYS_LOADED_180D >= 20                      THEN 'INVESTIGATE_STOPPED'
        ELSE                                                  'INVESTIGATE_MISS'
    END                                                                AS FINAL_ACTION
FROM enriched
;


/* ============================================================================
   VALIDATION CHECKLIST

   After running all three queries, verify:

   [ ] DAYS_LOADED_180D from Query 3 matches TOTAL_LOAD_DAYS_180D from Query 2
   [ ] LAST_LOAD_DT from Query 3 matches the top row's LOAD_DT from Query 1
   [ ] AVG_GAP_DAYS from Query 3 is close to ACTUAL_AVG_GAP from Query 2
   [ ] DETECTED_CADENCE matches the pattern you eyeballed in Query 1
   [ ] FINAL_ACTION matches your gut based on the actual pattern

   If everything lines up, we scale this to all tables in the schema.
   If something is off, we fix it here where it's easy to debug.
============================================================================ */
