
/* ============================================================================
   STEP 8 - MISSING TABLE LOAD DETECTION  (Single-Table Validation)
   ----------------------------------------------------------------------------
   WHAT THIS QUERY DOES:
     For ONE CT table, looks at its actual load history over the past 365 days
     and decides whether today's silence is normal (within the table's own
     pattern) or a real miss that needs investigation.

   HOW IT WORKS:
     Every table has its own natural rhythm.  The largest gap the table has
     ever had between consecutive loads (MAX_GAP_DAYS) is treated as the
     "normal maximum silence" for that table.  We give it a 50% buffer
     (multiply by 1.5) to allow for natural variation and call that the
     EXPECTED_WINDOW_DAYS.  If the current silence (DAYS_SINCE_LAST_LOAD)
     stays inside that window, it is EXPECTED_GAP.  If it exceeds the window
     by up to 3x, it is UNEXPECTED_MISS.  Beyond 3x, it is STOPPED_LOADING.

     Numeric example (rare table like TPLRT__CT):
       Loaded 4 times in past year with gaps of 63, 91, and 119 days
       MAX_GAP_DAYS      = 119
       EXPECTED_WINDOW   = 119 * 1.5 = 178.5 days
       Today's silence   = 30 days
       30 <= 178.5   ->  EXPECTED_GAP   ->  OK, no action

     Numeric example (daily table):
       Loaded every day for a year, MAX_GAP_DAYS = 1
       EXPECTED_WINDOW   = 1 * 1.5 = 1.5 days  (floored at 2 for safety)
       Today's silence   = 3 days
       3 > 2   ->  UNEXPECTED_MISS   ->  investigate

     Same formula, opposite verdicts, because each table is judged
     against its own history not a company-wide rule.

   WHY NOT USE INFORMATION_SCHEMA.LAST_ALTERED:
     Validation showed LAST_ALTERED can differ from real load activity by up
     to 214 days (e.g. TF_TB085__CT).  LAST_ALTERED is updated by clones,
     stats refreshes, and DDL, not just actual loads.  HEADER__TIMESTAMP is
     the only reliable source of when a real load happened.

   HOW TO USE:
     1. Update the FROM clause on line 88 to the target table
     2. Run the query
     3. Read the single result row against the guide at the bottom

   HOW TO READ THE RESULTS:
     LOAD_STATUS column tells you the verdict:
        LOADED_TODAY      -> table loaded today, no action needed
        EXPECTED_GAP      -> gap is within the table's normal pattern
        UNEXPECTED_MISS   -> gap exceeded normal, but still recent
                             (recent activity but missed today) - investigate
        STOPPED_LOADING   -> gap is 3x+ the normal - real problem
        NO_ACTIVITY       -> zero loads in 365 days - confirm with team
        INSUFFICIENT_HISTORY -> only 1 load found - not enough to judge yet

     MISSED_EXPECTED_LOAD_FLAG - Y if silence exceeded normal window
     STOPPED_LOADING_FLAG      - Y if silence exceeded 3x normal window

   WHAT TO DO WITH RESULTS:
     LOADED_TODAY / EXPECTED_GAP        -> no action, everything normal
     UNEXPECTED_MISS                    -> check Qlik/Talend/Airflow, may need incident
     STOPPED_LOADING                    -> escalate to Siddesh, likely incident
     NO_ACTIVITY                        -> confirm with team (may be by design)
     INSUFFICIENT_HISTORY               -> monitor, table too new to judge

   AUTHOR: Lokesh Varma
   REQUIREMENTS: Siddesh Gannu (Datamod Monitoring SOP r1, Step 8)
============================================================================ */


/* ---- Session setup ---- */
USE ROLE SF_PROD_DATA_ENGINEER;
USE WAREHOUSE PROD_DATA_ENGINEER_WH;
USE DATABASE PROD_LANDING;


WITH

/* Pull every distinct day the table had a real load in the past 365 days.
   BEFOREIMAGE rows are Qlik CDC bookkeeping and MUST be excluded - they are
   not real load activity. */
LOAD_DATES AS (
    SELECT DISTINCT
        TO_DATE(HEADER__TIMESTAMP)                                     AS LOAD_DATE
    FROM PROD_LANDING.ING.TZRAE__CT                                    -- (!) target table
    WHERE HEADER__TIMESTAMP IS NOT NULL
      AND TO_DATE(HEADER__TIMESTAMP) >= DATEADD(DAY, -365, CURRENT_DATE())
      AND UPPER(NVL(HEADER__OPERATION, '')) <> 'BEFOREIMAGE'
),

/* Compute the gap in days between each load and the previous load.
   The first row will have GAP_DAYS = NULL (nothing before it). */
GAPS AS (
    SELECT
        LOAD_DATE,
        DATEDIFF(
            DAY,
            LAG(LOAD_DATE) OVER (ORDER BY LOAD_DATE),
            LOAD_DATE
        )                                                              AS GAP_DAYS
    FROM LOAD_DATES
),

/* Aggregate the whole history into single-row stats for this table. */
TABLE_STATS AS (
    SELECT
        COUNT(*)                                                       AS ACTIVE_LOAD_DAYS,
        MAX(LOAD_DATE)                                                 AS LAST_LOAD_DATE,
        MAX(GAP_DAYS)                                                  AS MAX_GAP_DAYS
    FROM GAPS
),

/* Derive the alert thresholds and days-stale for classification.
   GREATEST(..., 2) enforces a minimum threshold of 2 days.  Without this,
   a truly-daily table would have EXPECTED_WINDOW = 1.5 and a legitimate
   3-day miss would slip through as EXPECTED_GAP. */
ANALYSIS AS (
    SELECT
        ACTIVE_LOAD_DAYS,
        LAST_LOAD_DATE,
        MAX_GAP_DAYS,

        GREATEST(ROUND(MAX_GAP_DAYS * 1.5, 2), 2)                      AS EXPECTED_WINDOW_DAYS,

        DATEDIFF(
            DAY,
            LAST_LOAD_DATE,
            CURRENT_DATE()
        )                                                              AS DAYS_SINCE_LAST_LOAD
    FROM TABLE_STATS
)

SELECT
    ACTIVE_LOAD_DAYS,                                                  -- number of distinct load days in last 365
    LAST_LOAD_DATE,                                                    -- most recent load date
    MAX_GAP_DAYS,                                                      -- biggest historical gap this table has had
    EXPECTED_WINDOW_DAYS,                                              -- how long is normal for THIS table to be silent
    DAYS_SINCE_LAST_LOAD,                                              -- how long has it actually been silent

    /* Main verdict.  Order of the WHEN clauses matters -
       edge cases must be caught FIRST or they will fall through. */
    CASE
        -- Case A: no loads at all in 365 days (Mary Beth's expired rule)
        WHEN ACTIVE_LOAD_DAYS = 0
            THEN 'NO_ACTIVITY'

        -- Case B: only 1 load found - cannot compute gaps
        WHEN ACTIVE_LOAD_DAYS = 1
            THEN 'INSUFFICIENT_HISTORY'

        -- Case C: loaded today - all good
        WHEN LAST_LOAD_DATE = CURRENT_DATE()
            THEN 'LOADED_TODAY'

        -- Case D: silence is within the table's normal pattern
        WHEN DAYS_SINCE_LAST_LOAD <= EXPECTED_WINDOW_DAYS
            THEN 'EXPECTED_GAP'

        -- Case E: silence exceeds normal but is less than 3x normal
        WHEN DAYS_SINCE_LAST_LOAD <= EXPECTED_WINDOW_DAYS * 3
            THEN 'UNEXPECTED_MISS'

        -- Case F: silence is 3x+ the normal window - table has stopped
        ELSE 'STOPPED_LOADING'
    END                                                                AS LOAD_STATUS,

    /* Flag Y if current silence exceeded the table's normal window. */
    CASE
        WHEN ACTIVE_LOAD_DAYS < 2                        THEN 'N/A'
        WHEN DAYS_SINCE_LAST_LOAD > EXPECTED_WINDOW_DAYS THEN 'Y'
        ELSE 'N'
    END                                                                AS MISSED_EXPECTED_LOAD_FLAG,

    /* Flag Y if current silence is 3x+ the normal window - real problem. */
    CASE
        WHEN ACTIVE_LOAD_DAYS < 2                                    THEN 'N/A'
        WHEN DAYS_SINCE_LAST_LOAD > EXPECTED_WINDOW_DAYS * 3          THEN 'Y'
        ELSE 'N'
    END                                                                AS STOPPED_LOADING_FLAG

FROM ANALYSIS
;


/* ============================================================================
   INTERPRETATION GUIDE

   Result row shows the verdict for the target table as of CURRENT_DATE.

   LOAD_STATUS values in order of severity:
     LOADED_TODAY           - table loaded today                       (OK)
     EXPECTED_GAP           - silence is normal for this table         (OK)
     INSUFFICIENT_HISTORY   - only 1 load in 365d, cannot judge         (WATCH)
     UNEXPECTED_MISS        - silence 1x-3x normal                     (INVESTIGATE)
     STOPPED_LOADING        - silence 3x+ normal                       (INVESTIGATE_STOPPED)
     NO_ACTIVITY            - zero loads in 365 days                   (CONFIRM_WITH_TEAM)

   VALIDATION CHECKLIST:
     [ ] MAX_GAP_DAYS matches what you see in the actual load history
     [ ] EXPECTED_WINDOW_DAYS is roughly 1.5x MAX_GAP_DAYS
     [ ] DAYS_SINCE_LAST_LOAD makes sense against LAST_LOAD_DATE
     [ ] LOAD_STATUS lines up with the gut check for this table's rhythm

   TESTING PLAN (validate on multiple patterns before scaling):
     1. A daily table  (expect DAILY / OK)
     2. A weekly table (expect WEEKLY-ish / OK when in cycle)
     3. A rare table like TPLRT__CT (expect EXPECTED_GAP / OK)
     4. A stopped table (expect STOPPED_LOADING / INVESTIGATE_STOPPED)
     5. A dormant table (expect NO_ACTIVITY / CONFIRM_WITH_TEAM)

   ONCE VALIDATED: swap the FROM clause and re-run per table, or wrap
   this in a schema-wide loop when ready to scale.
============================================================================ */
