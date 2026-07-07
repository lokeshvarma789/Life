/* ============================================================================
   STEP 9 - DAG MONITORING AND VALIDATION (FMC TABLE)
   ----------------------------------------------------------------------------
   WHAT THIS QUERY DOES:
     For each DAG that started today (ING, FRAT_ING, FRATDB), shows today's
     run with its runtime, success/failure/running/hung status, and compares
     the runtime against that DAG's own historical average using standard
     deviation - flagging CRITICAL (>3 STDDEV), WARNING (>2 STDDEV), or
     NORMAL, per Siddesh's specified method.

   HOW IT WORKS:
     FMC_LOAD_BATCH_HISTORY has one row per DAG run (batch). For each DAG we
     build a historical baseline from its own past successful runs over the
     last 90 days, EXCLUDING today so a DAG cannot skew its own comparison:
        AVG_RUNTIME_SECONDS    = average runtime of past successful runs
        STDDEV_RUNTIME_SECONDS = standard deviation of those runtimes
     Then today's runtime is compared against those bounds.

     Numeric example:
       DAG "FMC_30_ING_INCR" history: avg = 480 sec, stddev = 60 sec
       Today's run took 720 sec
       ABS(720 - 480) = 240 sec deviation
       3 * stddev = 180 sec  -> 240 > 180  -> CRITICAL

   CURRENTLY RUNNING vs HUNG:
     SUCCESS_FLAG is NULL while a DAG is still executing.
       - Started today, still NULL   -> CURRENTLY_RUNNING (in progress)
       - Started earlier, still NULL -> HUNG (never finished)

   SCOPE:
     Filtered to the 3 DAGs Siddesh's SOP calls out: ING, FRAT_ING, FRATDB.
     Change the BKCC filter below to widen scope.

   HOW TO USE:
     1. Adjust the BKCC filter below if you need a different schema set
     2. Run the query
     3. Read top rows first (sorted by severity)

   HOW TO READ THE RESULTS:
     RUN_STATUS column:
        SUCCESS            - completed normally
        FAILED             - SUCCESS_FLAG = 0
        HUNG               - started earlier, never finished
        CURRENTLY_RUNNING  - started today, in progress
        NO_RUN_TODAY       - baseline exists but no run today

     RUNTIME_STATUS column (only meaningful when RUN_STATUS = SUCCESS):
        CRITICAL             - runtime deviated by >3 STDDEV from history
        WARNING              - runtime deviated by 2-3 STDDEV
        NORMAL               - within 2 STDDEV
        INSUFFICIENT_HISTORY - fewer than 5 historical successful runs

     RUNTIME_DIRECTION column:
        LONG_RUNTIME    - today's run took longer than usual
        SHORT_RUNTIME   - today's run finished faster than usual
        WITHIN_RANGE    - inside normal band

   WHAT TO DO WITH RESULTS:
     FAILED / HUNG                   -> escalate to Siddesh, likely incident
     CRITICAL + LONG_RUNTIME         -> investigate immediately
     WARNING + LONG_RUNTIME          -> note for review, watch next run
     SHORT_RUNTIME                   -> check if data was actually processed
     NORMAL                          -> no action

   SAFETY:
     This query is READ-ONLY monitoring. Never run or recommend running any
     FMC_10 DAG - it performs a full initial load and will TRUNCATE the
     entire Raw Vault. Only Michael or Josh may authorize that.

   NOTE ON RECORD COUNTS:
     Siddesh confirmed that record-count comparison is not needed for this
     step - FMC does not store row counts, and adding that would require
     source-side counting and a separate history table. Runtime + status
     is sufficient for Step 9.

   AUTHOR: Lokesh Varma
   REQUIREMENTS: Siddesh Gannu (Datamod Monitoring SOP r1, Step 9)
============================================================================ */


/* ---- Session setup ---- */
USE ROLE SF_PROD_DATA_ENGINEER;
USE WAREHOUSE PROD_DATA_ENGINEER_WH;
USE DATABASE PROD_DV;


WITH

/* All batch runs for the DAGs in scope, past 90 days. Includes failed and
   still-running rows - we filter by SUCCESS_FLAG later where needed, not
   at the source, so failures and hangs stay visible. */
DAG_HISTORY AS (
    SELECT
        DAG_NAME,
        BKCC,
        DV_LOAD_BATCH_ID,
        LOAD_START_DATE,
        LOAD_END_DATE,
        SUCCESS_FLAG,
        DATEDIFF(SECOND, LOAD_START_DATE, LOAD_END_DATE) AS RUNTIME_SECONDS
    FROM PROD_DV.FMC.FMC_LOAD_BATCH_HISTORY
    WHERE BKCC IN ('ING', 'FRAT_ING', 'FRATDB')                        -- (!) adjust scope here
      AND LOAD_START_DATE >= DATEADD(DAY, -90, CURRENT_TIMESTAMP())
),

/* Baseline per DAG: only completed successful runs from BEFORE today.
   Excluding today prevents a DAG from being compared against itself. */
DAG_BASELINE AS (
    SELECT
        DAG_NAME,
        COUNT(*)                    AS HISTORICAL_RUN_COUNT,
        AVG(RUNTIME_SECONDS)        AS AVG_RUNTIME_SECONDS,
        STDDEV(RUNTIME_SECONDS)     AS STDDEV_RUNTIME_SECONDS,
        MIN(RUNTIME_SECONDS)        AS MIN_RUNTIME_SECONDS,
        MAX(RUNTIME_SECONDS)        AS MAX_RUNTIME_SECONDS
    FROM DAG_HISTORY
    WHERE SUCCESS_FLAG = 1
      AND DATE(LOAD_END_DATE) < CURRENT_DATE()
    GROUP BY DAG_NAME
),

/* Today's run per DAG. Filtering by CURRENT_DATE ensures we monitor what
   is happening TODAY, not the most recent run whenever it happened. */
TODAY_RUN AS (
    SELECT *
    FROM DAG_HISTORY
    WHERE DATE(LOAD_START_DATE) = CURRENT_DATE()
)

SELECT
    COALESCE(t.DAG_NAME, b.DAG_NAME)         AS DAG_NAME,
    t.BKCC,
    t.DV_LOAD_BATCH_ID,
    t.LOAD_START_DATE,
    t.LOAD_END_DATE,

    ROUND(t.RUNTIME_SECONDS / 60.0, 2)       AS RUNTIME_MINUTES,
    b.HISTORICAL_RUN_COUNT,
    ROUND(b.AVG_RUNTIME_SECONDS / 60.0, 2)   AS AVG_RUNTIME_MINUTES,
    ROUND(b.STDDEV_RUNTIME_SECONDS / 60.0, 2) AS STDDEV_RUNTIME_MINUTES,
    ROUND(b.MIN_RUNTIME_SECONDS / 60.0, 2)   AS MIN_RUNTIME_MINUTES,
    ROUND(b.MAX_RUNTIME_SECONDS / 60.0, 2)   AS MAX_RUNTIME_MINUTES,

    /* Overall run status - failed, hung, running, or completed. */
    CASE
        WHEN t.DAG_NAME IS NULL                                    THEN 'NO_RUN_TODAY'
        WHEN t.SUCCESS_FLAG = 1                                    THEN 'SUCCESS'
        WHEN t.SUCCESS_FLAG = 0                                    THEN 'FAILED'
        WHEN t.SUCCESS_FLAG IS NULL
             AND DATE(t.LOAD_START_DATE) = CURRENT_DATE()          THEN 'CURRENTLY_RUNNING'
        WHEN t.SUCCESS_FLAG IS NULL                                THEN 'HUNG'
        ELSE 'UNKNOWN'
    END                                       AS RUN_STATUS,

    /* Runtime classification vs historical band - only for successful runs. */
    CASE
        WHEN t.SUCCESS_FLAG <> 1 OR t.SUCCESS_FLAG IS NULL         THEN NULL
        WHEN b.STDDEV_RUNTIME_SECONDS IS NULL
             OR b.HISTORICAL_RUN_COUNT < 5                         THEN 'INSUFFICIENT_HISTORY'
        WHEN ABS(t.RUNTIME_SECONDS - b.AVG_RUNTIME_SECONDS)
             > (3 * b.STDDEV_RUNTIME_SECONDS)                      THEN 'CRITICAL'
        WHEN ABS(t.RUNTIME_SECONDS - b.AVG_RUNTIME_SECONDS)
             > (2 * b.STDDEV_RUNTIME_SECONDS)                      THEN 'WARNING'
        ELSE 'NORMAL'
    END                                       AS RUNTIME_STATUS,

    /* Direction of the deviation - was it long or short. */
    CASE
        WHEN t.SUCCESS_FLAG <> 1 OR t.SUCCESS_FLAG IS NULL         THEN NULL
        WHEN b.STDDEV_RUNTIME_SECONDS IS NULL                      THEN NULL
        WHEN t.RUNTIME_SECONDS
             > b.AVG_RUNTIME_SECONDS + (2 * b.STDDEV_RUNTIME_SECONDS) THEN 'LONG_RUNTIME'
        WHEN t.RUNTIME_SECONDS
             < b.AVG_RUNTIME_SECONDS - (2 * b.STDDEV_RUNTIME_SECONDS) THEN 'SHORT_RUNTIME'
        ELSE 'WITHIN_RANGE'
    END                                       AS RUNTIME_DIRECTION

FROM TODAY_RUN t
FULL OUTER JOIN DAG_BASELINE b ON t.DAG_NAME = b.DAG_NAME
ORDER BY
    CASE
        WHEN t.SUCCESS_FLAG = 0                                    THEN 1  -- failed first
        WHEN t.SUCCESS_FLAG IS NULL
             AND DATE(t.LOAD_START_DATE) <> CURRENT_DATE()         THEN 2  -- hung next
        WHEN t.DAG_NAME IS NULL                                    THEN 3  -- no run today
        WHEN ABS(t.RUNTIME_SECONDS - b.AVG_RUNTIME_SECONDS)
             > (3 * b.STDDEV_RUNTIME_SECONDS)                      THEN 4  -- CRITICAL
        WHEN ABS(t.RUNTIME_SECONDS - b.AVG_RUNTIME_SECONDS)
             > (2 * b.STDDEV_RUNTIME_SECONDS)                      THEN 5  -- WARNING
        ELSE 6
    END,
    COALESCE(t.DAG_NAME, b.DAG_NAME)
;

