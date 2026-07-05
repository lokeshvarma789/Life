/* ============================================================================
   STEP 8 - MISSING TABLE LOAD DETECTION
   ----------------------------------------------------------------------------
   WHAT THIS QUERY DOES:
     Returns every CT table in the target schema that has NOT been loaded
     today, along with how many days stale it is and a recommended action.

   HOW IT WORKS:
     Reads INFORMATION_SCHEMA.TABLES.LAST_ALTERED - Snowflake updates this
     column automatically whenever any DML (INSERT / UPDATE / DELETE /
     MERGE / COPY INTO) touches the table. Qlik replication and Talend
     jobs both trigger DML, so LAST_ALTERED reflects the most recent
     load activity for each table.

       LAST_ALTERED = today   ->  loaded today  (filtered out)
       LAST_ALTERED < today   ->  missing load  (returned)

     No table scans required - metadata-only, runs in seconds.

   HOW TO USE:
     1. Update the schema in the WHERE clause below       (marked with (!))
     2. Update the SCHEMA_LABEL literal in the SELECT     (marked with (!))
     3. Run.

   HOW TO READ THE RESULTS:
     Rows are sorted by priority - top rows need attention first.

     FINAL_ACTION column tells you what to do:

       INVESTIGATE               Missed today's load. Check now.
       INVESTIGATE_SHORT_STALE   Silent for 2-7 days. Real miss.
       INVESTIGATE_LONG_STALE    Silent for 8-30 days. Likely a problem.
       NEW_TABLE_MONITOR         Table < 7 days old. Watch, don't alert.
       REVIEW_IF_DORMANT         31-90 days silent. Confirm intentional.
       LIKELY_DORMANT_CONFIRM    90+ days silent. Probably by design.

     DAYS_STALE tells you the exact gap since last load.
     STALENESS_CATEGORY gives the human-readable bucket.

   WHAT TO DO WITH THE RESULTS:
     - Anything INVESTIGATE / INVESTIGATE_SHORT_STALE / INVESTIGATE_LONG_STALE
       -> Confirm with Siddesh, check Qlik/Talend/Airflow, raise incident
          if in PROD and unexpected.
     - NEW_TABLE_MONITOR -> Note it, watch it next day, no ticket needed.
     - REVIEW_IF_DORMANT / LIKELY_DORMANT_CONFIRM -> Log in Snowflake
       Issues Tracker "Observations" section so we stop re-investigating.

     For deeper detail on any flagged table, run Step 5 on it - Step 5
     scans HEADER__TIMESTAMP history and shows the actual load cadence.

   EDGE CASE TO KNOW:
     If Qlik or Talend ran successfully today but had ZERO changes to
     sync, the table stays untouched and shows here as "not loaded".
     Cross-check with Talend Management Console or Airflow FMC before
     escalating a table you know should be quiet-but-successful.

   AUTHOR: Lokesh Varma
   REQUIREMENTS: Siddesh Gannu (Datamod Monitoring SOP r1, Step 8)
============================================================================ */


/* ---- Session setup ---- */
USE ROLE SF_PROD_DATA_ENGINEER;
USE WAREHOUSE PROD_DATA_ENGINEER_WH;
USE DATABASE PROD_LANDING;


WITH

-- Pull every CT table in the target schema from the metadata catalog.
ct_inventory AS (
    SELECT
        TABLE_SCHEMA                                                   AS SCHEMA_NM,
        TABLE_NAME                                                     AS TABLE_NM,
        CREATED                                                        AS TABLE_CREATED_TS,
        LAST_ALTERED                                                   AS LAST_ACTIVITY_TS,
        ROW_COUNT                                                      AS TOTAL_ROWS,
        BYTES                                                          AS TOTAL_BYTES
    FROM PROD_LANDING.INFORMATION_SCHEMA.TABLES
    WHERE TABLE_SCHEMA   = 'ING'                                       -- (!) target schema
      AND TABLE_NAME LIKE '%\\_\\_CT' ESCAPE '\\'
      AND TABLE_TYPE     = 'BASE TABLE'
),

-- Derive dates, days-stale, and today-loaded flag.
enriched AS (
    SELECT
        SCHEMA_NM,
        TABLE_NM,
        TABLE_CREATED_TS,
        DATE(TABLE_CREATED_TS)                                         AS TABLE_CREATED_DT,
        DATEDIFF(DAY, TABLE_CREATED_TS, CURRENT_TIMESTAMP())           AS TABLE_AGE_DAYS,
        LAST_ACTIVITY_TS,
        DATE(LAST_ACTIVITY_TS)                                         AS LAST_LOAD_DT,
        DATEDIFF(DAY, DATE(LAST_ACTIVITY_TS), CURRENT_DATE())          AS DAYS_STALE,
        TOTAL_ROWS,
        TOTAL_BYTES,
        CASE
            WHEN DATE(LAST_ACTIVITY_TS) = CURRENT_DATE() THEN 'Y'
            ELSE 'N'
        END                                                            AS LOADED_TODAY_FLAG
    FROM ct_inventory
),

-- Bucket the staleness and decide the action.
classified AS (
    SELECT
        e.*,

        -- Staleness bucket (human-readable)
        CASE
            WHEN DAYS_STALE = 0                    THEN 'LOADED_TODAY'
            WHEN DAYS_STALE = 1                    THEN 'MISSED_TODAY'
            WHEN DAYS_STALE BETWEEN 2  AND 3       THEN 'SHORT_MISS_2_TO_3_DAYS'
            WHEN DAYS_STALE BETWEEN 4  AND 7       THEN 'WEEK_STALE'
            WHEN DAYS_STALE BETWEEN 8  AND 30      THEN 'MONTH_STALE'
            WHEN DAYS_STALE BETWEEN 31 AND 90      THEN 'VERY_STALE'
            WHEN DAYS_STALE > 90                   THEN 'LIKELY_DORMANT'
            ELSE                                        'UNKNOWN'
        END                                                            AS STALENESS_CATEGORY,

        -- Action to take
        CASE
            WHEN DAYS_STALE = 0                    THEN 'OK'
            WHEN TABLE_AGE_DAYS < 7                THEN 'NEW_TABLE_MONITOR'
            WHEN DAYS_STALE = 1                    THEN 'INVESTIGATE'
            WHEN DAYS_STALE BETWEEN 2  AND 7       THEN 'INVESTIGATE_SHORT_STALE'
            WHEN DAYS_STALE BETWEEN 8  AND 30      THEN 'INVESTIGATE_LONG_STALE'
            WHEN DAYS_STALE BETWEEN 31 AND 90      THEN 'REVIEW_IF_DORMANT'
            WHEN DAYS_STALE > 90                   THEN 'LIKELY_DORMANT_CONFIRM'
            ELSE                                        'UNKNOWN'
        END                                                            AS FINAL_ACTION
    FROM enriched e
)

-- Final output: all missing tables, highest priority first.
SELECT
    'ING'                              AS SCHEMA_LABEL,                -- (!) match target schema
    CURRENT_DATE()                     AS CHECK_DATE,
    CURRENT_TIMESTAMP()                AS CHECK_RUN_AT,
    SCHEMA_NM,
    TABLE_NM,
    LOADED_TODAY_FLAG,
    LAST_ACTIVITY_TS                   AS LAST_LOAD_TIMESTAMP,
    LAST_LOAD_DT,
    DAYS_STALE,
    STALENESS_CATEGORY,
    TABLE_CREATED_DT,
    TABLE_AGE_DAYS,
    TOTAL_ROWS,
    ROUND(TOTAL_BYTES / (1024.0 * 1024.0), 2)                          AS TOTAL_SIZE_MB,
    FINAL_ACTION
FROM classified
WHERE LOADED_TODAY_FLAG = 'N'
ORDER BY
    CASE FINAL_ACTION
        WHEN 'INVESTIGATE'              THEN 1
        WHEN 'INVESTIGATE_SHORT_STALE'  THEN 2
        WHEN 'INVESTIGATE_LONG_STALE'   THEN 3
        WHEN 'NEW_TABLE_MONITOR'        THEN 4
        WHEN 'REVIEW_IF_DORMANT'        THEN 5
        WHEN 'LIKELY_DORMANT_CONFIRM'   THEN 6
        ELSE                                 7
    END,
    DAYS_STALE DESC,
    TABLE_NM
;
