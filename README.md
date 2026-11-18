/* =====================================================================================
   DEV_TELEMETRY  -  Virtuous task telemetry (Epic 3 / US-3.3)
   -------------------------------------------------------------------------------------
   Author   : Lokesh Varma (draft prepared with senior review notes)
   Status   : DEV STARTER - NOT YET EXECUTED. Review with Michael before deploying.
              Expect small syntax fixes on first run; every object is DEV-scoped.
   Design   : Michael's "Detailed Telemetry Architecture" + "DEV_TELEMETRY Core Tables"
              + "Operational Views and Calculations" + "Monitoring and Alert Lifecycle".
              Michael's column names are kept exactly. Extra columns are marked PROPOSED.

   FLOW
     36 Virtuous load tasks (stream -> CT landing, each sets SYSTEM$SET_RETURN_VALUE)
        |
        v  every 15 min (5 min after the loads)
     TSK_TELEMETRY_COLLECT   -> SP_COLLECT_TASK_TELEMETRY  -> TASK_EXECUTION_HISTORY (append-only)
                                                            -> TASK_METRIC_HISTORY    (append-only)
        |  AFTER
     TSK_TELEMETRY_EVALUATE  -> SP_EVALUATE_TASK_HEALTH    -> ALERT_HISTORY (open / escalate / resolve)
                             -> SP_NOTIFY_ALERTS           -> email (once per alert)
     TSK_TELEMETRY_BASELINE  -> SP_COLLECT_DQ_RECON + SP_CALC_TASK_BASELINE (daily, versioned)
     ALR_TELEMETRY_WATCHDOG  -> independent Snowflake ALERT: fires if the telemetry graph stops

   PLACEHOLDERS - find/replace before running
     <LANDING_DB>.<LANDING_SCHEMA>   where the 36 streams, load tasks and __CT tables live
     <SHARE_DB>.<SHARE_SCHEMA>       the Virtuous data share
     <TELEMETRY_WH>                  XS warehouse for telemetry (or switch to serverless, see 11)
     <LOAD_WH>                       warehouse used by the load tasks
     <EMAIL_INTEGRATION>             notification integration (TYPE = EMAIL)
     <ALERT_RECIPIENTS>              a@kofc.org,b@kofc.org  (no quotes - it sits inside quotes)
     <ALLOWED_RECIPIENT_LIST>        'a@kofc.org', 'b@kofc.org'  (for the integration)
                                     Both must be VERIFIED emails of users in this Snowflake account.
     DEV_TELEMETRY.TELEMETRY         telemetry location. If DEV_TELEMETRY is a SCHEMA (not a
                                     database), replace with <DEV_DB>.DEV_TELEMETRY everywhere.

   RUN ORDER
     1 (admin)  -> 2 tables -> 3 UDFs -> 5..9 procedures -> 10 views -> 12 config seed
     -> 13 test kit (US-3.3 evidence) -> 11 tasks + watchdog (resume last)
     Section 4 is the standard for the load tasks (only if the loaders do not already
     publish row counts).

   CONVENTIONS
     TASK_NAME  = UPPER fully-qualified 'DB.SCHEMA.TASK' everywhere (config, history, alerts)
     Timestamps = TIMESTAMP_LTZ; all telemetry tasks run with TIMEZONE = America/New_York
     Facts (execution, metric, baseline) are append-only. ALERT_HISTORY is a lifecycle
     table: NOTIFIED_TIMESTAMP / STATUS / RESOLVED_TIMESTAMP are updated on the same row,
     exactly as Michael's columns imply.
   ===================================================================================== */


/* =====================================================================================
   1. DATABASE, SCHEMAS, ROLES, GRANTS   (send to admin - do not run with an elevated
      personal role; tasks must be owned by a functional role, not a person)
   ===================================================================================== */
USE ROLE SYSADMIN;
CREATE DATABASE IF NOT EXISTS DEV_TELEMETRY COMMENT = 'Pipeline telemetry (DEV)';
CREATE SCHEMA   IF NOT EXISTS DEV_TELEMETRY.TELEMETRY WITH MANAGED ACCESS
  COMMENT = 'Task execution facts, metrics, baselines, alerts';
CREATE SCHEMA   IF NOT EXISTS DEV_TELEMETRY.TELEMETRY_TEST
  COMMENT = 'US-3.3 failure-injection tasks. DEV only.';

USE ROLE SECURITYADMIN;
CREATE ROLE IF NOT EXISTS DEV_TELEMETRY_ENGINEER COMMENT = 'Owns and runs telemetry objects';
CREATE ROLE IF NOT EXISTS DEV_TELEMETRY_READER   COMMENT = 'Read-only telemetry views';
GRANT ROLE DEV_TELEMETRY_READER TO ROLE DEV_TELEMETRY_ENGINEER;

GRANT USAGE ON DATABASE DEV_TELEMETRY                 TO ROLE DEV_TELEMETRY_ENGINEER;
GRANT ALL   ON SCHEMA   DEV_TELEMETRY.TELEMETRY       TO ROLE DEV_TELEMETRY_ENGINEER;
GRANT ALL   ON SCHEMA   DEV_TELEMETRY.TELEMETRY_TEST  TO ROLE DEV_TELEMETRY_ENGINEER;
GRANT USAGE ON WAREHOUSE <TELEMETRY_WH>               TO ROLE DEV_TELEMETRY_ENGINEER;

-- Visibility into the Virtuous landing layer (SHOW TASKS needs a privilege on each task)
GRANT USAGE   ON DATABASE <LANDING_DB>                                TO ROLE DEV_TELEMETRY_ENGINEER;
GRANT USAGE   ON SCHEMA   <LANDING_DB>.<LANDING_SCHEMA>               TO ROLE DEV_TELEMETRY_ENGINEER;
GRANT MONITOR ON ALL    TASKS   IN SCHEMA <LANDING_DB>.<LANDING_SCHEMA> TO ROLE DEV_TELEMETRY_ENGINEER;
GRANT MONITOR ON FUTURE TASKS   IN SCHEMA <LANDING_DB>.<LANDING_SCHEMA> TO ROLE DEV_TELEMETRY_ENGINEER;
-- Reconciliation reads (SELECT on a stream does NOT advance its offset; only DML does)
GRANT SELECT  ON ALL    STREAMS IN SCHEMA <LANDING_DB>.<LANDING_SCHEMA> TO ROLE DEV_TELEMETRY_ENGINEER;
GRANT SELECT  ON FUTURE STREAMS IN SCHEMA <LANDING_DB>.<LANDING_SCHEMA> TO ROLE DEV_TELEMETRY_ENGINEER;
GRANT SELECT  ON ALL    TABLES  IN SCHEMA <LANDING_DB>.<LANDING_SCHEMA> TO ROLE DEV_TELEMETRY_ENGINEER;
GRANT SELECT  ON FUTURE TABLES  IN SCHEMA <LANDING_DB>.<LANDING_SCHEMA> TO ROLE DEV_TELEMETRY_ENGINEER;
GRANT IMPORTED PRIVILEGES ON DATABASE <SHARE_DB>                      TO ROLE DEV_TELEMETRY_ENGINEER;

-- Reader role: views only
GRANT USAGE  ON DATABASE DEV_TELEMETRY                        TO ROLE DEV_TELEMETRY_READER;
GRANT USAGE  ON SCHEMA   DEV_TELEMETRY.TELEMETRY              TO ROLE DEV_TELEMETRY_READER;
GRANT SELECT ON FUTURE VIEWS IN SCHEMA DEV_TELEMETRY.TELEMETRY TO ROLE DEV_TELEMETRY_READER;

USE ROLE ACCOUNTADMIN;   -- account-level privileges
GRANT EXECUTE TASK      ON ACCOUNT TO ROLE DEV_TELEMETRY_ENGINEER;
GRANT EXECUTE ALERT     ON ACCOUNT TO ROLE DEV_TELEMETRY_ENGINEER;
GRANT MONITOR EXECUTION ON ACCOUNT TO ROLE DEV_TELEMETRY_ENGINEER;   -- task history for all tasks
-- Email: recipients must be VERIFIED emails of users in this Snowflake account
CREATE NOTIFICATION INTEGRATION IF NOT EXISTS <EMAIL_INTEGRATION>
  TYPE = EMAIL
  ENABLED = TRUE
  ALLOWED_RECIPIENTS = (<ALLOWED_RECIPIENT_LIST>);
GRANT USAGE ON INTEGRATION <EMAIL_INTEGRATION> TO ROLE DEV_TELEMETRY_ENGINEER;

USE ROLE DEV_TELEMETRY_ENGINEER;
USE WAREHOUSE <TELEMETRY_WH>;
USE SCHEMA DEV_TELEMETRY.TELEMETRY;


/* =====================================================================================
   2. CORE TABLES  (Michael's 5 tables; PK/FK are informational - Snowflake does not
      enforce them, so every load below is idempotent by design)
   ===================================================================================== */

-- 2.1 One row per task run. Immutable execution facts. Terminal states only.
CREATE TABLE IF NOT EXISTS DEV_TELEMETRY.TELEMETRY.TASK_EXECUTION_HISTORY (
    TASK_RUN_ID          VARCHAR(32)    NOT NULL,  -- deterministic hash, see FN_TASK_RUN_ID
    TASK_NAME            VARCHAR(500)   NOT NULL,  -- DB.SCHEMA.TASK (upper)
    GRAPH_NAME           VARCHAR(500),             -- root task of the graph (= TASK_NAME if standalone)
    STATE                VARCHAR(30)    NOT NULL,  -- SUCCEEDED | FAILED | FAILED_AND_AUTO_SUSPENDED | CANCELLED | SKIPPED
    ERROR_CODE           VARCHAR(50),
    ERROR_MESSAGE        VARCHAR(4000),
    SCHEDULED_TIME       TIMESTAMP_LTZ,
    START_TIME           TIMESTAMP_LTZ,
    END_TIME             TIMESTAMP_LTZ,
    DURATION_SECONDS     NUMBER(18,3),
    ROWS_INSERTED        NUMBER(38,0),             -- NULL = loader did not report (never fake 0)
    ROWS_UPDATED         NUMBER(38,0),
    ROWS_DELETED         NUMBER(38,0),
    SOURCE_NAME          VARCHAR(500),             -- from task return value
    STREAM_NAME          VARCHAR(500),
    TARGET_NAME          VARCHAR(500),
    COLLECTED_TIMESTAMP  TIMESTAMP_LTZ  NOT NULL,
    -- PROPOSED additions
    QUERY_ID             VARCHAR(100),             -- TASK_HISTORY.QUERY_ID (may be NULL for SKIPPED)
    ROOT_TASK_ID         VARCHAR(100),
    GRAPH_RUN_ID         NUMBER(38,0),             -- TASK_HISTORY.RUN_ID (one per graph run)
    ATTEMPT_NUMBER       NUMBER(10,0),
    SCHEDULED_FROM       VARCHAR(30),              -- SCHEDULE | EXECUTE TASK | ...
    START_DELAY_SECONDS  NUMBER(18,3),             -- start - scheduled (warehouse queuing)
    RETURN_VALUE         VARCHAR(10000),           -- raw SYSTEM$SET_RETURN_VALUE payload
    CONSTRAINT PK_TASK_EXECUTION_HISTORY PRIMARY KEY (TASK_RUN_ID)
)
COMMENT = 'Append-only. One row per terminal task run collected from INFORMATION_SCHEMA.TASK_HISTORY.';

-- 2.2 One row per run + metric (also daily rollups and data-quality metrics)
CREATE TABLE IF NOT EXISTS DEV_TELEMETRY.TELEMETRY.TASK_METRIC_HISTORY (
    METRIC_HISTORY_ID    NUMBER(38,0)   IDENTITY(1,1),  -- PROPOSED surrogate key
    TASK_RUN_ID          VARCHAR(32),                   -- NULL for daily / table-level DQ metrics
    TASK_NAME            VARCHAR(500)   NOT NULL,
    METRIC_NAME          VARCHAR(100)   NOT NULL,       -- DURATION_SECONDS, ROWS_TOTAL, DAILY_ROWS_TOTAL, DQ_* ...
    METRIC_VALUE         NUMBER(38,6),
    UNIT_OF_MEASURE      VARCHAR(30),                   -- SECONDS | ROWS | PCT | COUNT
    DIMENSION_KEY        VARCHAR(200),                  -- RUN | DAY | TABLE | COLUMN:<name>
    OBSERVED_TIMESTAMP   TIMESTAMP_LTZ  NOT NULL,
    COLLECTED_TIMESTAMP  TIMESTAMP_LTZ  NOT NULL,
    CONSTRAINT PK_TASK_METRIC_HISTORY PRIMARY KEY (METRIC_HISTORY_ID),
    CONSTRAINT FK_TMH_TASK_RUN FOREIGN KEY (TASK_RUN_ID)
        REFERENCES DEV_TELEMETRY.TELEMETRY.TASK_EXECUTION_HISTORY (TASK_RUN_ID)
)
COMMENT = 'Append-only. Run metrics come from SUCCEEDED runs only, so the baseline is clean by construction.';

-- 2.3 Control metadata: what to monitor, limits, routing. One row per task + metric.
--     METRIC_NAME values and how the limits are read:
--       TASK_STATE             FAILED alerts                      (no limits)
--       SCHEDULE               OVERDUE + SUSPENDED alerts         limits = tolerance MINUTES beyond EXPECTED_FREQUENCY_MINUTES
--       DURATION_SECONDS       LONG_RUNNING                       ABSOLUTE seconds, or ZSCORE
--       ROWS_TOTAL             LOW_/HIGH_VOLUME per run           ZSCORE (|z| >= 2 warning, >= 3 critical)
--       DAILY_ROWS_TOTAL       LOW_/HIGH_VOLUME per day           ZSCORE, day-of-week segmented
--       STREAM_HOURS_TO_STALE  STREAM_STALE                       ABSOLUTE hours, BELOW
--       DQ_*                   DATA_QUALITY                       ABSOLUTE, direction configurable
CREATE TABLE IF NOT EXISTS DEV_TELEMETRY.TELEMETRY.TASK_MONITOR_CONFIG (
    TASK_NAME                   VARCHAR(500)  NOT NULL,
    METRIC_NAME                 VARCHAR(100)  NOT NULL,
    EXPECTED_FREQUENCY_MINUTES  NUMBER(10,0),
    WARNING_LIMIT               NUMBER(18,4),
    CRITICAL_LIMIT              NUMBER(18,4),            -- for ZSCORE also the k in mean +/- k*stddev
    BASELINE_WINDOW_DAYS        NUMBER(5,0)   DEFAULT 180,
    MIN_SAMPLE_COUNT            NUMBER(10,0)  DEFAULT 30,
    DAY_OF_WEEK_SEGMENT         BOOLEAN       DEFAULT FALSE,
    MONITOR_ENABLED             BOOLEAN       DEFAULT TRUE,   -- evaluate + record alerts
    ALERT_ENABLED               BOOLEAN       DEFAULT TRUE,   -- send notifications (FALSE = shadow mode)
    -- PROPOSED additions
    LIMIT_TYPE                  VARCHAR(10)   DEFAULT 'ABSOLUTE',  -- ABSOLUTE | ZSCORE
    BREACH_DIRECTION            VARCHAR(10)   DEFAULT 'ABOVE',     -- ABOVE | BELOW | BOTH
    ALERT_ROUTE                 VARCHAR(200),                      -- e.g. EMAIL:DATA_SUPPORT, SNOW:<group>
    UPDATED_BY                  VARCHAR(200),
    UPDATED_TIMESTAMP           TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP(),
    CONSTRAINT PK_TASK_MONITOR_CONFIG PRIMARY KEY (TASK_NAME, METRIC_NAME)
)
COMMENT = 'Monitoring control metadata. Change limits here - never in code.';

-- 2.4 Calculated statistics. Append-only versions: every alert can be reproduced with
--     the baseline that was active at detection time (BASELINE_ID stored on the alert).
CREATE TABLE IF NOT EXISTS DEV_TELEMETRY.TELEMETRY.TASK_METRIC_BASELINE (
    BASELINE_ID           NUMBER(38,0)  IDENTITY(1,1),   -- PROPOSED
    TASK_NAME             VARCHAR(500)  NOT NULL,
    METRIC_NAME           VARCHAR(100)  NOT NULL,
    SEGMENT_KEY           VARCHAR(20)   NOT NULL,        -- ALL | DOW_1 .. DOW_7 (ISO, Monday = 1)
    WINDOW_START          TIMESTAMP_LTZ,
    WINDOW_END            TIMESTAMP_LTZ,
    SAMPLE_COUNT          NUMBER(10,0),
    MEAN_VALUE            NUMBER(38,6),
    STDDEV_VALUE          NUMBER(38,6),
    LOWER_LIMIT           NUMBER(38,6),
    UPPER_LIMIT           NUMBER(38,6),
    CALCULATED_TIMESTAMP  TIMESTAMP_LTZ NOT NULL,
    CONSTRAINT PK_TASK_METRIC_BASELINE PRIMARY KEY (BASELINE_ID)
)
COMMENT = 'Append-only daily baseline versions (rolling window from config).';

-- 2.5 Alert lifecycle. One OPEN row per DEDUP_KEY at a time.
CREATE TABLE IF NOT EXISTS DEV_TELEMETRY.TELEMETRY.ALERT_HISTORY (
    ALERT_ID            NUMBER(38,0)   IDENTITY(1,1),
    TASK_RUN_ID         VARCHAR(32),
    TASK_NAME           VARCHAR(500)   NOT NULL,
    METRIC_NAME         VARCHAR(100),
    ALERT_TYPE          VARCHAR(30)    NOT NULL,  -- FAILED | LONG_RUNNING | OVERDUE | LOW_VOLUME | HIGH_VOLUME
                                                  -- | DATA_QUALITY | SUSPENDED* | STREAM_STALE* | METRIC_BREACH*  (*PROPOSED)
    SEVERITY            VARCHAR(10)    NOT NULL,  -- WARNING | CRITICAL
    OBSERVED_VALUE      NUMBER(38,6),
    EXPECTED_VALUE      NUMBER(38,6),
    Z_SCORE             NUMBER(18,4),
    DETECTED_TIMESTAMP  TIMESTAMP_LTZ  NOT NULL,
    NOTIFIED_TIMESTAMP  TIMESTAMP_LTZ,
    STATUS              VARCHAR(15)    NOT NULL,  -- OPEN | RESOLVED | ESCALATED
    RESOLVED_TIMESTAMP  TIMESTAMP_LTZ,
    -- PROPOSED additions
    DEDUP_KEY           VARCHAR(1000)  NOT NULL,  -- TASK|ALERT_TYPE|METRIC|DIMENSION
    DIMENSION_KEY       VARCHAR(200),
    BASELINE_ID         NUMBER(38,0),
    ALERT_MESSAGE       VARCHAR(4000),
    ALERT_ROUTE         VARCHAR(200),
    CONSTRAINT PK_ALERT_HISTORY PRIMARY KEY (ALERT_ID)
)
COMMENT = 'Alert lifecycle: insert once per dedup key, notify once, resolve when the condition clears.';


/* =====================================================================================
   3. HELPER FUNCTIONS  (single source of truth for the run key and severity logic)
   ===================================================================================== */

-- Deterministic run key. Uses epoch nanoseconds so it does not depend on session timezone
-- (a timezone-dependent string would create duplicate "new" runs).
CREATE OR REPLACE FUNCTION DEV_TELEMETRY.TELEMETRY.FN_TASK_RUN_ID(
    TASK_FQN VARCHAR, RUN_ID NUMBER, ATTEMPT_NUMBER NUMBER, SCHEDULED_TIME TIMESTAMP_LTZ)
RETURNS VARCHAR
AS
$$
    MD5(UPPER(TASK_FQN) || '|' ||
        COALESCE(TO_VARCHAR(RUN_ID), '') || '|' ||
        COALESCE(TO_VARCHAR(ATTEMPT_NUMBER), '') || '|' ||
        COALESCE(TO_VARCHAR(DATE_PART(EPOCH_NANOSECOND, SCHEDULED_TIME)), ''))
$$;

-- Severity for a value against WARNING/CRITICAL limits.
--   ZSCORE  : V is a z-score. ABOVE z >= k, BELOW z <= -k, BOTH |z| >= k   (slide: |Z| >= 2 / 3)
--   ABSOLUTE: V is the raw value. ABOVE V > limit, BELOW V < limit
CREATE OR REPLACE FUNCTION DEV_TELEMETRY.TELEMETRY.FN_BREACH_SEVERITY(
    V FLOAT, WARN_LIMIT FLOAT, CRIT_LIMIT FLOAT, LIMIT_TYPE VARCHAR, DIRECTION VARCHAR)
RETURNS VARCHAR
AS
$$
    CASE
      WHEN V IS NULL THEN NULL
      WHEN LIMIT_TYPE = 'ZSCORE' THEN
        CASE
          WHEN CRIT_LIMIT IS NOT NULL AND (
               (DIRECTION IN ('ABOVE', 'BOTH') AND V >=  CRIT_LIMIT) OR
               (DIRECTION IN ('BELOW', 'BOTH') AND V <= -CRIT_LIMIT)) THEN 'CRITICAL'
          WHEN WARN_LIMIT IS NOT NULL AND (
               (DIRECTION IN ('ABOVE', 'BOTH') AND V >=  WARN_LIMIT) OR
               (DIRECTION IN ('BELOW', 'BOTH') AND V <= -WARN_LIMIT)) THEN 'WARNING'
        END
      ELSE
        CASE
          WHEN CRIT_LIMIT IS NOT NULL AND (
               (DIRECTION = 'BELOW' AND V < CRIT_LIMIT) OR
               (DIRECTION <> 'BELOW' AND V > CRIT_LIMIT)) THEN 'CRITICAL'
          WHEN WARN_LIMIT IS NOT NULL AND (
               (DIRECTION = 'BELOW' AND V < WARN_LIMIT) OR
               (DIRECTION <> 'BELOW' AND V > WARN_LIMIT)) THEN 'WARNING'
        END
    END
$$;


/* =====================================================================================
   4. LOAD TASK STANDARD  (one per Virtuous table, 36 total)
   -------------------------------------------------------------------------------------
   Why: QUERY_HISTORY shows 0 rows for a CALL / scripting block (the counts sit on child
   statements), so per-run row counts must be PUBLISHED by the loader. Each load task sets
   a JSON return value that the collector reads from TASK_HISTORY.RETURN_VALUE.
   I/U/D are BUSINESS operations from stream metadata, not DML counts: the CT table is
   insert-only and an update arrives as DELETE+INSERT pair with METADATA$ISUPDATE = TRUE.
   Per-table placeholders: <TABLE>, <COLUMN_LIST> (use the generator at 4.3).
   ===================================================================================== */

-- 4.1 Stream on the shared table. Requires the provider to enable CHANGE_TRACKING on the
--     shared table. NEVER "CREATE OR REPLACE" a stream in a deploy script - it resets the
--     offset and silently loses or re-delivers changes.
CREATE STREAM IF NOT EXISTS <LANDING_DB>.<LANDING_SCHEMA>.STR_VIRTUOUS_<TABLE>
  ON TABLE <SHARE_DB>.<SHARE_SCHEMA>.<TABLE>
  SHOW_INITIAL_ROWS = TRUE   -- first consumption returns the current table as INSERTs (initial load)
  COMMENT = 'Virtuous CDC stream for <TABLE>';

-- 4.2 Load task
CREATE TASK IF NOT EXISTS <LANDING_DB>.<LANDING_SCHEMA>.TSK_LOAD_VIRTUOUS_<TABLE>
  WAREHOUSE = <LOAD_WH>
  SCHEDULE = '15 MINUTE'
  ALLOW_OVERLAPPING_EXECUTION = FALSE
  USER_TASK_TIMEOUT_MS = 3600000
  SUSPEND_TASK_AFTER_NUM_FAILURES = 3
  COMMENT = 'Virtuous <TABLE>: stream -> __CT landing. Publishes row counts via SYSTEM$SET_RETURN_VALUE.'
  WHEN SYSTEM$STREAM_HAS_DATA('<LANDING_DB>.<LANDING_SCHEMA>.STR_VIRTUOUS_<TABLE>')
AS
EXECUTE IMMEDIATE $$
DECLARE
  v_ins    NUMBER DEFAULT 0;
  v_upd    NUMBER DEFAULT 0;
  v_del    NUMBER DEFAULT 0;
  v_loaded NUMBER DEFAULT 0;
  v_ret    VARCHAR;
  e_count_mismatch EXCEPTION (-20101, 'Loaded row count does not match stream change counts');
BEGIN
  BEGIN TRANSACTION;
  -- Repeatable read: inside one transaction the stream returns the same rows to every
  -- statement, so these counts describe exactly the rows inserted below.
  SELECT COUNT_IF(METADATA$ACTION = 'INSERT' AND NOT METADATA$ISUPDATE),
         COUNT_IF(METADATA$ACTION = 'INSERT' AND METADATA$ISUPDATE),
         COUNT_IF(METADATA$ACTION = 'DELETE' AND NOT METADATA$ISUPDATE)
    INTO :v_ins, :v_upd, :v_del
    FROM <LANDING_DB>.<LANDING_SCHEMA>.STR_VIRTUOUS_<TABLE>;

  INSERT INTO <LANDING_DB>.<LANDING_SCHEMA>.<TABLE>__CT (<COLUMN_LIST>, HEADER__OPERATION, HEADER__TIMESTAMP)
  SELECT <COLUMN_LIST>,
         CASE WHEN METADATA$ACTION = 'DELETE' THEN 'DELETE'
              WHEN METADATA$ISUPDATE          THEN 'UPDATE'
              ELSE 'INSERT' END,
         CURRENT_TIMESTAMP()
  FROM <LANDING_DB>.<LANDING_SCHEMA>.STR_VIRTUOUS_<TABLE>
  WHERE NOT (METADATA$ACTION = 'DELETE' AND METADATA$ISUPDATE);   -- drop update before-images
  v_loaded := SQLROWCOUNT;

  IF (v_loaded <> v_ins + v_upd + v_del) THEN
    RAISE e_count_mismatch;          -- built-in data-quality guard; rolls back, offset not advanced
  END IF;
  COMMIT;                            -- stream offset advances here

  SELECT TO_JSON(OBJECT_CONSTRUCT(
           'inserted', :v_ins, 'updated', :v_upd, 'deleted', :v_del,
           'source',   '<SHARE_DB>.<SHARE_SCHEMA>.<TABLE>',
           'stream',   '<LANDING_DB>.<LANDING_SCHEMA>.STR_VIRTUOUS_<TABLE>',
           'target',   '<LANDING_DB>.<LANDING_SCHEMA>.<TABLE>__CT'))
    INTO :v_ret;
  CALL SYSTEM$SET_RETURN_VALUE(:v_ret);
EXCEPTION
  WHEN OTHER THEN
    ROLLBACK;
    RAISE;
END;
$$;

-- 4.3 Column-list generator for the 36 tables. Quoted names keep case - SaaS shares often
--     use mixed-case column names, and an unquoted list would fail or map wrongly.
SELECT TABLE_NAME,
       LISTAGG('"' || COLUMN_NAME || '"', ', ') WITHIN GROUP (ORDER BY ORDINAL_POSITION) AS COLUMN_LIST
FROM <SHARE_DB>.INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = '<SHARE_SCHEMA>'
GROUP BY TABLE_NAME
ORDER BY TABLE_NAME;


/* =====================================================================================
   5. COLLECTOR  -  SP_COLLECT_TASK_TELEMETRY
   -------------------------------------------------------------------------------------
   - Reads INFORMATION_SCHEMA.TASK_HISTORY (near real time, 7-day max window).
     ACCOUNT_USAGE.TASK_HISTORY has ~45 min latency - too slow for a 15-min monitor.
   - Terminal states only (immutable facts). RUNNING is read live by the evaluator.
   - Overlapping lookback + anti-join on TASK_RUN_ID instead of a "max timestamp"
     watermark: late-completing runs are never skipped, re-runs never duplicate.
   - History + metrics are written in one transaction.
   ===================================================================================== */
CREATE OR REPLACE PROCEDURE DEV_TELEMETRY.TELEMETRY.SP_COLLECT_TASK_TELEMETRY(
    P_DATABASE       VARCHAR,
    P_SCHEMA         VARCHAR,
    P_LOOKBACK_HOURS NUMBER DEFAULT 6)
RETURNS VARCHAR
LANGUAGE SQL
EXECUTE AS CALLER
AS
$$
DECLARE
  v_db       VARCHAR;
  v_sch      VARCHAR;
  v_prefix   VARCHAR;
  v_hours    NUMBER;
  v_ts       TIMESTAMP_LTZ;
  v_sql      VARCHAR;
  v_raw_rows NUMBER DEFAULT 0;
  v_runs     NUMBER DEFAULT 0;
  v_metrics  NUMBER DEFAULT 0;
  v_warning  VARCHAR DEFAULT '';
  e_bad_input EXCEPTION (-20001, 'SP_COLLECT_TASK_TELEMETRY: invalid database or schema identifier');
BEGIN
  v_db     := UPPER(TRIM(P_DATABASE));
  v_sch    := UPPER(TRIM(P_SCHEMA));
  v_prefix := v_db || '.' || v_sch || '.';
  v_hours  := LEAST(GREATEST(ROUND(COALESCE(P_LOOKBACK_HOURS, 6)), 1), 168);
  v_ts     := CURRENT_TIMESTAMP();

  -- identifiers are concatenated into dynamic SQL below, so validate them first
  IF (NOT REGEXP_LIKE(v_db, '^[A-Z0-9_]+$') OR NOT REGEXP_LIKE(v_sch, '^[A-Z0-9_]+$')) THEN
    RAISE e_bad_input;
  END IF;

  -- 1. Raw task history for the whole database (RESULT_LIMIT applies before any filter)
  v_sql := 'CREATE OR REPLACE TEMPORARY TABLE DEV_TELEMETRY.TELEMETRY.TMP_COL_TH AS ' ||
           'SELECT * FROM TABLE(' || v_db || '.INFORMATION_SCHEMA.TASK_HISTORY(' ||
           'SCHEDULED_TIME_RANGE_START => DATEADD(''hour'', -' || TO_VARCHAR(v_hours) || ', CURRENT_TIMESTAMP()), ' ||
           'RESULT_LIMIT => 10000))';
  EXECUTE IMMEDIATE :v_sql;

  SELECT COUNT(*) INTO :v_raw_rows FROM DEV_TELEMETRY.TELEMETRY.TMP_COL_TH;
  IF (v_raw_rows >= 10000) THEN
    v_warning := 'RESULT_LIMIT reached - older runs in the window may be missing; lower P_LOOKBACK_HOURS';
  END IF;

  -- 2. Task id -> name map (to resolve GRAPH_NAME from ROOT_TASK_ID)
  v_sql := 'SHOW TASKS IN DATABASE ' || v_db;
  EXECUTE IMMEDIATE :v_sql;
  CREATE OR REPLACE TEMPORARY TABLE DEV_TELEMETRY.TELEMETRY.TMP_COL_TASKS AS
    SELECT "id"::VARCHAR                                                   AS TASK_ID,
           UPPER("database_name" || '.' || "schema_name" || '.' || "name") AS TASK_FQN
    FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));

  BEGIN TRANSACTION;

  -- 3. Append unseen terminal runs
  INSERT INTO DEV_TELEMETRY.TELEMETRY.TASK_EXECUTION_HISTORY (
      TASK_RUN_ID, TASK_NAME, GRAPH_NAME, STATE, ERROR_CODE, ERROR_MESSAGE,
      SCHEDULED_TIME, START_TIME, END_TIME, DURATION_SECONDS,
      ROWS_INSERTED, ROWS_UPDATED, ROWS_DELETED, SOURCE_NAME, STREAM_NAME, TARGET_NAME,
      COLLECTED_TIMESTAMP, QUERY_ID, ROOT_TASK_ID, GRAPH_RUN_ID, ATTEMPT_NUMBER,
      SCHEDULED_FROM, START_DELAY_SECONDS, RETURN_VALUE)
  WITH th AS (
      SELECT t.*,
             UPPER(t.DATABASE_NAME || '.' || t.SCHEMA_NAME || '.' || t.NAME) AS TASK_FQN,
             TRY_PARSE_JSON(t.RETURN_VALUE)                                   AS RV
      FROM DEV_TELEMETRY.TELEMETRY.TMP_COL_TH t
      WHERE UPPER(t.SCHEMA_NAME) = :v_sch
        AND t.STATE IN ('SUCCEEDED', 'FAILED', 'FAILED_AND_AUTO_SUSPENDED', 'CANCELLED', 'SKIPPED')
  ),
  runs AS (
      SELECT DEV_TELEMETRY.TELEMETRY.FN_TASK_RUN_ID(th.TASK_FQN, th.RUN_ID, th.ATTEMPT_NUMBER, th.SCHEDULED_TIME)
                                                               AS TASK_RUN_ID,
             th.TASK_FQN                                       AS TASK_NAME,
             COALESCE(g.TASK_FQN, th.TASK_FQN)                 AS GRAPH_NAME,
             th.STATE,
             TO_VARCHAR(th.ERROR_CODE)                         AS ERROR_CODE,
             LEFT(th.ERROR_MESSAGE, 4000)                      AS ERROR_MESSAGE,
             th.SCHEDULED_TIME,
             th.QUERY_START_TIME                               AS START_TIME,
             th.COMPLETED_TIME                                 AS END_TIME,
             DATEDIFF('millisecond', th.QUERY_START_TIME, th.COMPLETED_TIME) / 1000 AS DURATION_SECONDS,
             TRY_TO_NUMBER(th.RV:inserted::VARCHAR)            AS ROWS_INSERTED,
             TRY_TO_NUMBER(th.RV:updated::VARCHAR)             AS ROWS_UPDATED,
             TRY_TO_NUMBER(th.RV:deleted::VARCHAR)             AS ROWS_DELETED,
             th.RV:source::VARCHAR                             AS SOURCE_NAME,   -- kept as published (may be case-sensitive)
             th.RV:stream::VARCHAR                             AS STREAM_NAME,
             th.RV:target::VARCHAR                             AS TARGET_NAME,
             th.QUERY_ID,
             th.ROOT_TASK_ID::VARCHAR                          AS ROOT_TASK_ID,
             TRY_TO_NUMBER(TO_VARCHAR(th.RUN_ID))              AS GRAPH_RUN_ID,
             th.ATTEMPT_NUMBER,
             th.SCHEDULED_FROM,
             DATEDIFF('millisecond', th.SCHEDULED_TIME, th.QUERY_START_TIME) / 1000 AS START_DELAY_SECONDS,
             LEFT(th.RETURN_VALUE, 10000)                      AS RETURN_VALUE
      FROM th
      LEFT JOIN DEV_TELEMETRY.TELEMETRY.TMP_COL_TASKS g
             ON g.TASK_ID = th.ROOT_TASK_ID::VARCHAR
  )
  SELECT r.TASK_RUN_ID, r.TASK_NAME, r.GRAPH_NAME, r.STATE, r.ERROR_CODE, r.ERROR_MESSAGE,
         r.SCHEDULED_TIME, r.START_TIME, r.END_TIME, r.DURATION_SECONDS,
         r.ROWS_INSERTED, r.ROWS_UPDATED, r.ROWS_DELETED, r.SOURCE_NAME, r.STREAM_NAME, r.TARGET_NAME,
         :v_ts, r.QUERY_ID, r.ROOT_TASK_ID, r.GRAPH_RUN_ID, r.ATTEMPT_NUMBER,
         r.SCHEDULED_FROM, r.START_DELAY_SECONDS, r.RETURN_VALUE
  FROM runs r
  WHERE NOT EXISTS (
          SELECT 1
          FROM DEV_TELEMETRY.TELEMETRY.TASK_EXECUTION_HISTORY h
          WHERE h.TASK_RUN_ID = r.TASK_RUN_ID
            AND h.SCHEDULED_TIME >= DATEADD('hour', -(:v_hours + 24), :v_ts))   -- pruning
  QUALIFY ROW_NUMBER() OVER (PARTITION BY r.TASK_RUN_ID ORDER BY r.END_TIME DESC NULLS LAST) = 1;
  v_runs := SQLROWCOUNT;

  -- 4. Metrics for the SUCCEEDED runs collected in THIS call (NULL counts are skipped, not zeroed)
  INSERT INTO DEV_TELEMETRY.TELEMETRY.TASK_METRIC_HISTORY (
      TASK_RUN_ID, TASK_NAME, METRIC_NAME, METRIC_VALUE, UNIT_OF_MEASURE,
      DIMENSION_KEY, OBSERVED_TIMESTAMP, COLLECTED_TIMESTAMP)
  SELECT TASK_RUN_ID, TASK_NAME, METRIC_NAME, METRIC_VALUE, UNIT_OF_MEASURE,
         'RUN', END_TIME, :v_ts
  FROM (
      SELECT r.TASK_RUN_ID, r.TASK_NAME, r.END_TIME, m.METRIC_NAME, m.UNIT_OF_MEASURE,
             CASE m.METRIC_NAME
               WHEN 'DURATION_SECONDS'    THEN r.DURATION_SECONDS
               WHEN 'START_DELAY_SECONDS' THEN r.START_DELAY_SECONDS
               WHEN 'ROWS_INSERTED'       THEN r.ROWS_INSERTED
               WHEN 'ROWS_UPDATED'        THEN r.ROWS_UPDATED
               WHEN 'ROWS_DELETED'        THEN r.ROWS_DELETED
               WHEN 'ROWS_TOTAL'          THEN r.ROWS_INSERTED + r.ROWS_UPDATED + r.ROWS_DELETED
             END AS METRIC_VALUE
      FROM DEV_TELEMETRY.TELEMETRY.TASK_EXECUTION_HISTORY r
      CROSS JOIN (SELECT COLUMN1 AS METRIC_NAME, COLUMN2 AS UNIT_OF_MEASURE
                  FROM VALUES ('DURATION_SECONDS',    'SECONDS'),
                              ('START_DELAY_SECONDS', 'SECONDS'),
                              ('ROWS_INSERTED',       'ROWS'),
                              ('ROWS_UPDATED',        'ROWS'),
                              ('ROWS_DELETED',        'ROWS'),
                              ('ROWS_TOTAL',          'ROWS')) m
      WHERE r.COLLECTED_TIMESTAMP = :v_ts
        AND r.STATE = 'SUCCEEDED'
        AND STARTSWITH(r.TASK_NAME, :v_prefix)
  )
  WHERE METRIC_VALUE IS NOT NULL;
  v_metrics := SQLROWCOUNT;

  COMMIT;

  RETURN 'collected runs=' || v_runs || ', metrics=' || v_metrics ||
         ', raw_history_rows=' || v_raw_rows ||
         IFF(v_warning = '', '', ' | WARNING: ' || v_warning);
EXCEPTION
  WHEN OTHER THEN
    ROLLBACK;
    RAISE;
END;
$$;


/* =====================================================================================
   6. BASELINE  -  SP_CALC_TASK_BASELINE  (daily)
   -------------------------------------------------------------------------------------
   Step 1: DAILY_ROWS_TOTAL rollup (includes ZERO days - the real low-volume signal for
           stream-triggered loads, where a run only happens when there is data).
   Step 2: Append a new baseline version per task / metric / segment.
           Eligibility (slide): successful runs only (by construction), minimum sample
           count, stddev > 0, optional day-of-week segment.
   ===================================================================================== */
CREATE OR REPLACE PROCEDURE DEV_TELEMETRY.TELEMETRY.SP_CALC_TASK_BASELINE()
RETURNS VARCHAR
LANGUAGE SQL
EXECUTE AS CALLER
AS
$$
DECLARE
  v_ts    TIMESTAMP_LTZ;
  v_daily NUMBER DEFAULT 0;
  v_base  NUMBER DEFAULT 0;
BEGIN
  v_ts := CURRENT_TIMESTAMP();

  -- 1. Daily rollup for the last 7 complete days (idempotent, self-healing)
  INSERT INTO DEV_TELEMETRY.TELEMETRY.TASK_METRIC_HISTORY (
      TASK_RUN_ID, TASK_NAME, METRIC_NAME, METRIC_VALUE, UNIT_OF_MEASURE,
      DIMENSION_KEY, OBSERVED_TIMESTAMP, COLLECTED_TIMESTAMP)
  WITH days AS (
      SELECT DATEADD('day', -ROW_NUMBER() OVER (ORDER BY SEQ4()), CURRENT_DATE()) AS D
      FROM TABLE(GENERATOR(ROWCOUNT => 7))
  ),
  cfg AS (
      SELECT TASK_NAME
      FROM DEV_TELEMETRY.TELEMETRY.TASK_MONITOR_CONFIG
      WHERE METRIC_NAME = 'DAILY_ROWS_TOTAL' AND MONITOR_ENABLED
  ),
  first_seen AS (   -- no zero-days before the task ever ran
      SELECT TASK_NAME, MIN(TO_DATE(SCHEDULED_TIME)) AS FIRST_DAY
      FROM DEV_TELEMETRY.TELEMETRY.TASK_EXECUTION_HISTORY
      WHERE TASK_NAME IN (SELECT TASK_NAME FROM cfg)
      GROUP BY TASK_NAME
  ),
  tot AS (
      SELECT TASK_NAME, TO_DATE(OBSERVED_TIMESTAMP) AS D, SUM(METRIC_VALUE) AS V
      FROM DEV_TELEMETRY.TELEMETRY.TASK_METRIC_HISTORY
      WHERE METRIC_NAME = 'ROWS_TOTAL' AND DIMENSION_KEY = 'RUN'
        AND OBSERVED_TIMESTAMP >= DATEADD('day', -8, CURRENT_DATE())
      GROUP BY TASK_NAME, TO_DATE(OBSERVED_TIMESTAMP)
  )
  SELECT NULL, c.TASK_NAME, 'DAILY_ROWS_TOTAL', COALESCE(t.V, 0), 'ROWS',
         'DAY', d.D::TIMESTAMP_LTZ, :v_ts
  FROM cfg c
  JOIN first_seen f ON f.TASK_NAME = c.TASK_NAME
  CROSS JOIN days d
  LEFT JOIN tot t ON t.TASK_NAME = c.TASK_NAME AND t.D = d.D
  WHERE d.D > f.FIRST_DAY                               -- first day is partial
    AND NOT EXISTS (
          SELECT 1 FROM DEV_TELEMETRY.TELEMETRY.TASK_METRIC_HISTORY m
          WHERE m.TASK_NAME = c.TASK_NAME
            AND m.METRIC_NAME = 'DAILY_ROWS_TOTAL'
            AND m.DIMENSION_KEY = 'DAY'
            AND TO_DATE(m.OBSERVED_TIMESTAMP) = d.D);
  v_daily := SQLROWCOUNT;

  -- 2. New baseline version (only for metrics configured as ZSCORE)
  INSERT INTO DEV_TELEMETRY.TELEMETRY.TASK_METRIC_BASELINE (
      TASK_NAME, METRIC_NAME, SEGMENT_KEY, WINDOW_START, WINDOW_END, SAMPLE_COUNT,
      MEAN_VALUE, STDDEV_VALUE, LOWER_LIMIT, UPPER_LIMIT, CALCULATED_TIMESTAMP)
  WITH obs AS (
      SELECT m.TASK_NAME, m.METRIC_NAME, m.METRIC_VALUE,
             IFF(c.DAY_OF_WEEK_SEGMENT, 'DOW_' || DAYOFWEEKISO(m.OBSERVED_TIMESTAMP), 'ALL') AS SEGMENT_KEY,
             DATEADD('day', -c.BASELINE_WINDOW_DAYS, :v_ts) AS WINDOW_START,
             COALESCE(c.MIN_SAMPLE_COUNT, 30)               AS MIN_SAMPLE_COUNT,
             COALESCE(c.CRITICAL_LIMIT, 3)                  AS K
      FROM DEV_TELEMETRY.TELEMETRY.TASK_METRIC_HISTORY m
      JOIN DEV_TELEMETRY.TELEMETRY.TASK_MONITOR_CONFIG c
        ON c.TASK_NAME = m.TASK_NAME AND c.METRIC_NAME = m.METRIC_NAME
      WHERE c.MONITOR_ENABLED
        AND c.LIMIT_TYPE = 'ZSCORE'
        AND m.METRIC_VALUE IS NOT NULL
        AND m.OBSERVED_TIMESTAMP >= DATEADD('day', -c.BASELINE_WINDOW_DAYS, :v_ts)
        AND m.OBSERVED_TIMESTAMP <  :v_ts
  )
  SELECT TASK_NAME, METRIC_NAME, SEGMENT_KEY,
         MIN(WINDOW_START), :v_ts, COUNT(*),
         AVG(METRIC_VALUE),
         STDDEV_SAMP(METRIC_VALUE),
         GREATEST(0, AVG(METRIC_VALUE) - MAX(K) * STDDEV_SAMP(METRIC_VALUE)),
         AVG(METRIC_VALUE) + MAX(K) * STDDEV_SAMP(METRIC_VALUE),
         :v_ts
  FROM obs
  GROUP BY TASK_NAME, METRIC_NAME, SEGMENT_KEY
  HAVING COUNT(*) >= MAX(MIN_SAMPLE_COUNT)
     AND STDDEV_SAMP(METRIC_VALUE) > 0;
  v_base := SQLROWCOUNT;

  RETURN 'daily_rollups=' || v_daily || ', baseline_versions=' || v_base;
END;
$$;


/* =====================================================================================
   7. DATA QUALITY  -  SP_COLLECT_DQ_RECON  (daily)
   -------------------------------------------------------------------------------------
   US-3.3 goal: "defects visible without manual reconciliation". PK-free reconciliation:
       source rows  =  net CT rows (INSERT +1, DELETE -1)  +  net rows still in the stream
   One statement, so all three reads share one snapshot. A lossless pipeline gives 0.
   Assumes the CT table holds full history from the initial load (SHOW_INITIAL_ROWS) and
   is never purged. Source/stream/target come from the load tasks' return values.
   ===================================================================================== */
CREATE OR REPLACE PROCEDURE DEV_TELEMETRY.TELEMETRY.SP_COLLECT_DQ_RECON(
    P_DATABASE VARCHAR,
    P_SCHEMA   VARCHAR)
RETURNS VARCHAR
LANGUAGE SQL
EXECUTE AS CALLER
AS
$$
DECLARE
  v_prefix  VARCHAR;
  v_ts      TIMESTAMP_LTZ;
  v_task    VARCHAR;
  v_src     VARCHAR;
  v_str     VARCHAR;
  v_tgt     VARCHAR;
  v_src_cnt NUMBER;
  v_net     NUMBER;
  v_ok      NUMBER DEFAULT 0;
  v_err     NUMBER DEFAULT 0;
  rs        RESULTSET;
BEGIN
  v_prefix := UPPER(TRIM(P_DATABASE)) || '.' || UPPER(TRIM(P_SCHEMA)) || '.';
  v_ts     := CURRENT_TIMESTAMP();

  rs := (SELECT h.TASK_NAME, h.SOURCE_NAME, h.STREAM_NAME, h.TARGET_NAME
         FROM DEV_TELEMETRY.TELEMETRY.TASK_EXECUTION_HISTORY h
         JOIN DEV_TELEMETRY.TELEMETRY.TASK_MONITOR_CONFIG c
           ON c.TASK_NAME = h.TASK_NAME AND c.METRIC_NAME = 'DQ_RECON_DIFF' AND c.MONITOR_ENABLED
         WHERE STARTSWITH(h.TASK_NAME, :v_prefix)
           AND h.SOURCE_NAME IS NOT NULL AND h.STREAM_NAME IS NOT NULL AND h.TARGET_NAME IS NOT NULL
           AND h.SCHEDULED_TIME >= DATEADD('day', -35, :v_ts)
         QUALIFY ROW_NUMBER() OVER (PARTITION BY h.TASK_NAME ORDER BY h.SCHEDULED_TIME DESC) = 1);
  LET c1 CURSOR FOR rs;

  FOR rec IN c1 DO
    v_task := rec.TASK_NAME;
    v_src  := rec.SOURCE_NAME;
    v_str  := rec.STREAM_NAME;
    v_tgt  := rec.TARGET_NAME;
    BEGIN
      SELECT (SELECT COUNT(*) FROM IDENTIFIER(:v_src)),
             (SELECT COALESCE(SUM(CASE HEADER__OPERATION WHEN 'INSERT' THEN 1
                                                         WHEN 'DELETE' THEN -1 ELSE 0 END), 0)
              FROM IDENTIFIER(:v_tgt))
           + (SELECT COUNT_IF(METADATA$ACTION = 'INSERT' AND NOT METADATA$ISUPDATE)
                   - COUNT_IF(METADATA$ACTION = 'DELETE' AND NOT METADATA$ISUPDATE)
              FROM IDENTIFIER(:v_str))
        INTO :v_src_cnt, :v_net;

      INSERT INTO DEV_TELEMETRY.TELEMETRY.TASK_METRIC_HISTORY (
          TASK_RUN_ID, TASK_NAME, METRIC_NAME, METRIC_VALUE, UNIT_OF_MEASURE,
          DIMENSION_KEY, OBSERVED_TIMESTAMP, COLLECTED_TIMESTAMP)
      SELECT NULL, :v_task, 'DQ_RECON_DIFF',  ABS(:v_src_cnt - :v_net), 'ROWS', 'TABLE', :v_ts, :v_ts
      UNION ALL
      SELECT NULL, :v_task, 'DQ_SOURCE_ROWS', :v_src_cnt,               'ROWS', 'TABLE', :v_ts, :v_ts
      UNION ALL
      SELECT NULL, :v_task, 'DQ_RECON_CHECK_ERROR', 0,                  'COUNT','TABLE', :v_ts, :v_ts;
      v_ok := v_ok + 1;
    EXCEPTION
      WHEN OTHER THEN   -- one bad table must not stop the other 35
        INSERT INTO DEV_TELEMETRY.TELEMETRY.TASK_METRIC_HISTORY (
            TASK_RUN_ID, TASK_NAME, METRIC_NAME, METRIC_VALUE, UNIT_OF_MEASURE,
            DIMENSION_KEY, OBSERVED_TIMESTAMP, COLLECTED_TIMESTAMP)
        SELECT NULL, :v_task, 'DQ_RECON_CHECK_ERROR', 1, 'COUNT', 'TABLE', :v_ts, :v_ts;
        v_err := v_err + 1;
    END;
  END FOR;

  RETURN 'recon tables ok=' || v_ok || ', errors=' || v_err;
END;
$$;


/* =====================================================================================
   8. HEALTH MONITOR  -  SP_EVALUATE_TASK_HEALTH
   -------------------------------------------------------------------------------------
   Deterministic rules first (never wait for a baseline - slide guardrail):
     FAILED        latest run (incl. SKIPPED) is FAILED / FAILED_AND_AUTO_SUSPENDED (CRITICAL),
                   CANCELLED (WARNING). A later SUCCEEDED or SKIPPED run = recovered.
     SUSPENDED     task suspended but has a SCHEDULE monitor              (PROPOSED type)
     OVERDUE       no SUCCEEDED/SKIPPED run within frequency + tolerance; suppressed while
                   suspended or executing (other alerts already cover those)
     LONG_RUNNING  live: still EXECUTING past the limit (read live from TASK_HISTORY)
     STREAM_STALE  stream stale / missing, or hours-to-stale below limit (PROPOSED type)
   Generic metric rule (latest observation per task/metric/dimension):
     ABSOLUTE limits -> DATA_QUALITY / LONG_RUNNING / volume
     ZSCORE vs the baseline active at observation time (ASOF JOIN)
        -> LOW_VOLUME / HIGH_VOLUME / LONG_RUNNING / METRIC_BREACH
   Lifecycle: one OPEN alert per DEDUP_KEY; WARNING->CRITICAL closes the warning as
   ESCALATED and opens a CRITICAL; alerts whose condition cleared become RESOLVED.
   ===================================================================================== */
CREATE OR REPLACE PROCEDURE DEV_TELEMETRY.TELEMETRY.SP_EVALUATE_TASK_HEALTH(
    P_DATABASE VARCHAR,
    P_SCHEMA   VARCHAR)
RETURNS VARCHAR
LANGUAGE SQL
EXECUTE AS CALLER
AS
$$
DECLARE
  v_db        VARCHAR;
  v_sch       VARCHAR;
  v_prefix    VARCHAR;
  v_ts        TIMESTAMP_LTZ;
  v_sql       VARCHAR;
  v_cand      NUMBER DEFAULT 0;
  v_escalated NUMBER DEFAULT 0;
  v_opened    NUMBER DEFAULT 0;
  v_resolved  NUMBER DEFAULT 0;
  e_bad_input EXCEPTION (-20002, 'SP_EVALUATE_TASK_HEALTH: invalid database or schema identifier');
BEGIN
  v_db     := UPPER(TRIM(P_DATABASE));
  v_sch    := UPPER(TRIM(P_SCHEMA));
  v_prefix := v_db || '.' || v_sch || '.';
  v_ts     := CURRENT_TIMESTAMP();

  IF (NOT REGEXP_LIKE(v_db, '^[A-Z0-9_]+$') OR NOT REGEXP_LIKE(v_sch, '^[A-Z0-9_]+$')) THEN
    RAISE e_bad_input;
  END IF;

  -- ---------- live state ----------
  v_sql := 'SHOW TASKS IN SCHEMA ' || v_db || '.' || v_sch;
  EXECUTE IMMEDIATE :v_sql;
  CREATE OR REPLACE TEMPORARY TABLE DEV_TELEMETRY.TELEMETRY.TMP_EVA_TASK_STATE AS
    SELECT UPPER("database_name" || '.' || "schema_name" || '.' || "name") AS TASK_NAME,
           "state"::VARCHAR    AS TASK_STATE,
           "schedule"::VARCHAR AS SCHEDULE
    FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));

  v_sql := 'CREATE OR REPLACE TEMPORARY TABLE DEV_TELEMETRY.TELEMETRY.TMP_EVA_EXECUTING AS ' ||
           'SELECT UPPER(DATABASE_NAME || ''.'' || SCHEMA_NAME || ''.'' || NAME) AS TASK_NAME, ' ||
           'DEV_TELEMETRY.TELEMETRY.FN_TASK_RUN_ID(UPPER(DATABASE_NAME || ''.'' || SCHEMA_NAME || ''.'' || NAME), ' ||
           'RUN_ID, ATTEMPT_NUMBER, SCHEDULED_TIME) AS TASK_RUN_ID, ' ||
           'QUERY_START_TIME, DATEDIFF(''second'', QUERY_START_TIME, CURRENT_TIMESTAMP()) AS ELAPSED_SECONDS ' ||
           'FROM TABLE(' || v_db || '.INFORMATION_SCHEMA.TASK_HISTORY(' ||
           'SCHEDULED_TIME_RANGE_START => DATEADD(''hour'', -24, CURRENT_TIMESTAMP()), RESULT_LIMIT => 10000)) ' ||
           'WHERE STATE = ''EXECUTING'' AND UPPER(SCHEMA_NAME) = ''' || v_sch || '''';
  EXECUTE IMMEDIATE :v_sql;

  v_sql := 'SHOW STREAMS IN DATABASE ' || v_db;
  EXECUTE IMMEDIATE :v_sql;
  CREATE OR REPLACE TEMPORARY TABLE DEV_TELEMETRY.TELEMETRY.TMP_EVA_STREAMS AS
    SELECT UPPER("database_name" || '.' || "schema_name" || '.' || "name") AS STREAM_NAME,
           "stale"::VARCHAR                                                 AS STALE,
           "stale_after"::TIMESTAMP_LTZ                                     AS STALE_AFTER
    FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));

  CREATE OR REPLACE TEMPORARY TABLE DEV_TELEMETRY.TELEMETRY.TMP_EVA_CANDIDATES (
      TASK_RUN_ID    VARCHAR,
      TASK_NAME      VARCHAR,
      METRIC_NAME    VARCHAR,
      DIMENSION_KEY  VARCHAR,
      ALERT_TYPE     VARCHAR,
      SEVERITY       VARCHAR,
      OBSERVED_VALUE NUMBER(38,6),
      EXPECTED_VALUE NUMBER(38,6),
      Z_SCORE        NUMBER(18,4),
      BASELINE_ID    NUMBER(38,0),
      ALERT_MESSAGE  VARCHAR(4000));

  -- ---------- FAILED ----------
  INSERT INTO DEV_TELEMETRY.TELEMETRY.TMP_EVA_CANDIDATES
      (TASK_RUN_ID, TASK_NAME, METRIC_NAME, DIMENSION_KEY, ALERT_TYPE, SEVERITY,
       OBSERVED_VALUE, EXPECTED_VALUE, Z_SCORE, BASELINE_ID, ALERT_MESSAGE)
  WITH last_run AS (
      SELECT h.*
      FROM DEV_TELEMETRY.TELEMETRY.TASK_EXECUTION_HISTORY h
      WHERE STARTSWITH(h.TASK_NAME, :v_prefix)
        AND h.SCHEDULED_TIME >= DATEADD('day', -35, :v_ts)
      QUALIFY ROW_NUMBER() OVER (PARTITION BY h.TASK_NAME
                                 ORDER BY h.SCHEDULED_TIME DESC, h.END_TIME DESC NULLS LAST) = 1
  )
  SELECT lr.TASK_RUN_ID, lr.TASK_NAME, 'TASK_STATE', 'RUN', 'FAILED',
         IFF(lr.STATE = 'CANCELLED', 'WARNING', 'CRITICAL'),
         NULL, NULL, NULL, NULL,
         LEFT(lr.STATE || ' at ' || COALESCE(TO_VARCHAR(lr.END_TIME, 'YYYY-MM-DD HH24:MI:SS'), '?') ||
              ' | ' || COALESCE(lr.ERROR_CODE, '') || ' ' || COALESCE(lr.ERROR_MESSAGE, ''), 4000)
  FROM last_run lr
  JOIN DEV_TELEMETRY.TELEMETRY.TASK_MONITOR_CONFIG c
    ON c.TASK_NAME = lr.TASK_NAME AND c.METRIC_NAME = 'TASK_STATE' AND c.MONITOR_ENABLED
  WHERE lr.STATE IN ('FAILED', 'FAILED_AND_AUTO_SUSPENDED', 'CANCELLED');

  -- ---------- SUSPENDED ----------
  INSERT INTO DEV_TELEMETRY.TELEMETRY.TMP_EVA_CANDIDATES
      (TASK_RUN_ID, TASK_NAME, METRIC_NAME, DIMENSION_KEY, ALERT_TYPE, SEVERITY,
       OBSERVED_VALUE, EXPECTED_VALUE, Z_SCORE, BASELINE_ID, ALERT_MESSAGE)
  SELECT NULL, c.TASK_NAME, 'SCHEDULE', 'TASK', 'SUSPENDED', 'CRITICAL',
         NULL, c.EXPECTED_FREQUENCY_MINUTES, NULL, NULL,
         'Task is SUSPENDED but is monitored as scheduled (expected every ' ||
         COALESCE(TO_VARCHAR(c.EXPECTED_FREQUENCY_MINUTES), '?') || ' min)'
  FROM DEV_TELEMETRY.TELEMETRY.TASK_MONITOR_CONFIG c
  JOIN DEV_TELEMETRY.TELEMETRY.TMP_EVA_TASK_STATE s ON s.TASK_NAME = c.TASK_NAME
  WHERE c.METRIC_NAME = 'SCHEDULE'
    AND c.MONITOR_ENABLED
    AND STARTSWITH(c.TASK_NAME, :v_prefix)
    AND LOWER(s.TASK_STATE) = 'suspended';

  -- ---------- OVERDUE ----------
  INSERT INTO DEV_TELEMETRY.TELEMETRY.TMP_EVA_CANDIDATES
      (TASK_RUN_ID, TASK_NAME, METRIC_NAME, DIMENSION_KEY, ALERT_TYPE, SEVERITY,
       OBSERVED_VALUE, EXPECTED_VALUE, Z_SCORE, BASELINE_ID, ALERT_MESSAGE)
  WITH last_ok AS (
      SELECT TASK_NAME, MAX(COALESCE(END_TIME, SCHEDULED_TIME)) AS LAST_OK_TIME
      FROM DEV_TELEMETRY.TELEMETRY.TASK_EXECUTION_HISTORY
      WHERE STARTSWITH(TASK_NAME, :v_prefix)
        AND STATE IN ('SUCCEEDED', 'SKIPPED')   -- SKIPPED = ran on schedule, stream empty
        AND SCHEDULED_TIME >= DATEADD('day', -35, :v_ts)
      GROUP BY TASK_NAME
  ),
  x AS (
      SELECT c.TASK_NAME,
             c.EXPECTED_FREQUENCY_MINUTES AS FREQ,
             c.WARNING_LIMIT, c.CRITICAL_LIMIT,
             l.LAST_OK_TIME,
             s.TASK_NAME AS FOUND_TASK,
             DATEDIFF('minute', COALESCE(l.LAST_OK_TIME, c.UPDATED_TIMESTAMP), :v_ts) AS MINUTES_SINCE
      FROM DEV_TELEMETRY.TELEMETRY.TASK_MONITOR_CONFIG c
      LEFT JOIN last_ok l                                    ON l.TASK_NAME = c.TASK_NAME
      LEFT JOIN DEV_TELEMETRY.TELEMETRY.TMP_EVA_TASK_STATE s ON s.TASK_NAME = c.TASK_NAME
      LEFT JOIN DEV_TELEMETRY.TELEMETRY.TMP_EVA_EXECUTING  r ON r.TASK_NAME = c.TASK_NAME
      WHERE c.METRIC_NAME = 'SCHEDULE'
        AND c.MONITOR_ENABLED
        AND STARTSWITH(c.TASK_NAME, :v_prefix)
        AND c.EXPECTED_FREQUENCY_MINUTES IS NOT NULL
        AND COALESCE(LOWER(s.TASK_STATE), 'missing') <> 'suspended'
        AND r.TASK_NAME IS NULL
  ),
  y AS (
      SELECT x.*,
             CASE
               WHEN CRITICAL_LIMIT IS NOT NULL AND MINUTES_SINCE > FREQ + CRITICAL_LIMIT THEN 'CRITICAL'
               WHEN WARNING_LIMIT  IS NOT NULL AND MINUTES_SINCE > FREQ + WARNING_LIMIT  THEN 'WARNING'
               WHEN WARNING_LIMIT IS NULL AND CRITICAL_LIMIT IS NULL AND MINUTES_SINCE > FREQ THEN 'CRITICAL'
             END AS SEVERITY
      FROM x
  )
  SELECT NULL, TASK_NAME, 'SCHEDULE', 'TASK', 'OVERDUE', SEVERITY,
         MINUTES_SINCE, FREQ, NULL, NULL,
         'No successful/skipped run for ' || MINUTES_SINCE || ' min (expected every ' || FREQ || ' min)' ||
         IFF(LAST_OK_TIME IS NULL, ' - no qualifying run since monitoring was configured', '') ||
         IFF(FOUND_TASK IS NULL, ' - TASK NOT FOUND (dropped or renamed?)', '')
  FROM y
  WHERE SEVERITY IS NOT NULL;

  -- ---------- LONG_RUNNING (live) ----------
  INSERT INTO DEV_TELEMETRY.TELEMETRY.TMP_EVA_CANDIDATES
      (TASK_RUN_ID, TASK_NAME, METRIC_NAME, DIMENSION_KEY, ALERT_TYPE, SEVERITY,
       OBSERVED_VALUE, EXPECTED_VALUE, Z_SCORE, BASELINE_ID, ALERT_MESSAGE)
  WITH cur_base AS (
      SELECT *
      FROM DEV_TELEMETRY.TELEMETRY.TASK_METRIC_BASELINE
      WHERE METRIC_NAME = 'DURATION_SECONDS'
      QUALIFY ROW_NUMBER() OVER (PARTITION BY TASK_NAME, METRIC_NAME, SEGMENT_KEY
                                 ORDER BY CALCULATED_TIMESTAMP DESC, BASELINE_ID DESC) = 1
  ),
  x AS (
      SELECT r.TASK_RUN_ID, r.TASK_NAME, r.ELAPSED_SECONDS,
             c.LIMIT_TYPE, c.WARNING_LIMIT, c.CRITICAL_LIMIT,
             b.BASELINE_ID, b.MEAN_VALUE,
             (r.ELAPSED_SECONDS - b.MEAN_VALUE) / NULLIF(b.STDDEV_VALUE, 0) AS Z_SCORE
      FROM DEV_TELEMETRY.TELEMETRY.TMP_EVA_EXECUTING r
      JOIN DEV_TELEMETRY.TELEMETRY.TASK_MONITOR_CONFIG c
        ON c.TASK_NAME = r.TASK_NAME AND c.METRIC_NAME = 'DURATION_SECONDS' AND c.MONITOR_ENABLED
      LEFT JOIN cur_base b
        ON b.TASK_NAME = r.TASK_NAME
       AND b.SEGMENT_KEY = IFF(c.DAY_OF_WEEK_SEGMENT, 'DOW_' || DAYOFWEEKISO(:v_ts), 'ALL')
  ),
  y AS (
      SELECT x.*,
             DEV_TELEMETRY.TELEMETRY.FN_BREACH_SEVERITY(
                 IFF(LIMIT_TYPE = 'ZSCORE', Z_SCORE, ELAPSED_SECONDS),
                 WARNING_LIMIT, CRITICAL_LIMIT, LIMIT_TYPE, 'ABOVE') AS SEVERITY
      FROM x
  )
  SELECT TASK_RUN_ID, TASK_NAME, 'DURATION_SECONDS', 'RUN', 'LONG_RUNNING', SEVERITY,
         ELAPSED_SECONDS,
         IFF(LIMIT_TYPE = 'ZSCORE', MEAN_VALUE, IFF(SEVERITY = 'CRITICAL', CRITICAL_LIMIT, WARNING_LIMIT)),
         ROUND(Z_SCORE, 4), BASELINE_ID,
         'Still EXECUTING after ' || ELAPSED_SECONDS || ' s' ||
         IFF(LIMIT_TYPE = 'ZSCORE', ' (z=' || TO_VARCHAR(ROUND(Z_SCORE, 2)) || ')', '')
  FROM y
  WHERE SEVERITY IS NOT NULL;

  -- ---------- STREAM_STALE ----------
  INSERT INTO DEV_TELEMETRY.TELEMETRY.TMP_EVA_CANDIDATES
      (TASK_RUN_ID, TASK_NAME, METRIC_NAME, DIMENSION_KEY, ALERT_TYPE, SEVERITY,
       OBSERVED_VALUE, EXPECTED_VALUE, Z_SCORE, BASELINE_ID, ALERT_MESSAGE)
  WITH task_stream AS (
      SELECT TASK_NAME, UPPER(STREAM_NAME) AS STREAM_NAME
      FROM DEV_TELEMETRY.TELEMETRY.TASK_EXECUTION_HISTORY
      WHERE STARTSWITH(TASK_NAME, :v_prefix)
        AND STREAM_NAME IS NOT NULL
        AND SCHEDULED_TIME >= DATEADD('day', -35, :v_ts)
      QUALIFY ROW_NUMBER() OVER (PARTITION BY TASK_NAME ORDER BY SCHEDULED_TIME DESC) = 1
  ),
  x AS (
      SELECT ts.TASK_NAME, ts.STREAM_NAME, st.STALE, st.STALE_AFTER,
             DATEDIFF('hour', :v_ts, st.STALE_AFTER) AS HOURS_TO_STALE,
             c.WARNING_LIMIT, c.CRITICAL_LIMIT
      FROM task_stream ts
      JOIN DEV_TELEMETRY.TELEMETRY.TASK_MONITOR_CONFIG c
        ON c.TASK_NAME = ts.TASK_NAME AND c.METRIC_NAME = 'STREAM_HOURS_TO_STALE' AND c.MONITOR_ENABLED
      LEFT JOIN DEV_TELEMETRY.TELEMETRY.TMP_EVA_STREAMS st ON st.STREAM_NAME = ts.STREAM_NAME
  ),
  y AS (
      SELECT x.*,
             CASE WHEN STALE IS NULL OR LOWER(STALE) = 'true' THEN 'CRITICAL'
                  ELSE DEV_TELEMETRY.TELEMETRY.FN_BREACH_SEVERITY(
                           HOURS_TO_STALE, WARNING_LIMIT, CRITICAL_LIMIT, 'ABSOLUTE', 'BELOW')
             END AS SEVERITY
      FROM x
  )
  SELECT NULL, TASK_NAME, 'STREAM_HOURS_TO_STALE', 'STREAM', 'STREAM_STALE', SEVERITY,
         HOURS_TO_STALE, IFF(SEVERITY = 'CRITICAL', CRITICAL_LIMIT, WARNING_LIMIT), NULL, NULL,
         CASE WHEN STALE IS NULL THEN 'Stream ' || STREAM_NAME || ' NOT FOUND (dropped or no privilege)'
              WHEN LOWER(STALE) = 'true' THEN 'Stream ' || STREAM_NAME || ' is STALE - full reload required'
              ELSE 'Stream ' || STREAM_NAME || ' goes stale in ' || HOURS_TO_STALE || ' h (at ' ||
                   TO_VARCHAR(STALE_AFTER, 'YYYY-MM-DD HH24:MI') || ')'
         END
  FROM y
  WHERE SEVERITY IS NOT NULL;

  -- ---------- generic metric rule: ABSOLUTE + ZSCORE ----------
  INSERT INTO DEV_TELEMETRY.TELEMETRY.TMP_EVA_CANDIDATES
      (TASK_RUN_ID, TASK_NAME, METRIC_NAME, DIMENSION_KEY, ALERT_TYPE, SEVERITY,
       OBSERVED_VALUE, EXPECTED_VALUE, Z_SCORE, BASELINE_ID, ALERT_MESSAGE)
  WITH cfg AS (
      SELECT *
      FROM DEV_TELEMETRY.TELEMETRY.TASK_MONITOR_CONFIG
      WHERE MONITOR_ENABLED
        AND STARTSWITH(TASK_NAME, :v_prefix)
        AND METRIC_NAME NOT IN ('TASK_STATE', 'SCHEDULE', 'STREAM_HOURS_TO_STALE')
  ),
  lm AS (   -- latest observation per task / metric / dimension (stale after 8 days = not evaluated)
      SELECT m.TASK_RUN_ID, m.TASK_NAME, m.METRIC_NAME, m.DIMENSION_KEY, m.METRIC_VALUE, m.OBSERVED_TIMESTAMP,
             IFF(c.DAY_OF_WEEK_SEGMENT, 'DOW_' || DAYOFWEEKISO(m.OBSERVED_TIMESTAMP), 'ALL') AS SEGMENT_KEY,
             c.LIMIT_TYPE, c.BREACH_DIRECTION, c.WARNING_LIMIT, c.CRITICAL_LIMIT
      FROM DEV_TELEMETRY.TELEMETRY.TASK_METRIC_HISTORY m
      JOIN cfg c ON c.TASK_NAME = m.TASK_NAME AND c.METRIC_NAME = m.METRIC_NAME
      WHERE m.OBSERVED_TIMESTAMP >= DATEADD('day', -8, :v_ts)
      QUALIFY ROW_NUMBER() OVER (PARTITION BY m.TASK_NAME, m.METRIC_NAME, m.DIMENSION_KEY
                                 ORDER BY m.OBSERVED_TIMESTAMP DESC, m.METRIC_HISTORY_ID DESC) = 1
  ),
  scored AS (   -- baseline that was active when the value was observed (reproducible alerts)
      SELECT lm.*, b.BASELINE_ID, b.MEAN_VALUE, b.STDDEV_VALUE,
             (lm.METRIC_VALUE - b.MEAN_VALUE) / NULLIF(b.STDDEV_VALUE, 0) AS Z_SCORE
      FROM lm
      ASOF JOIN DEV_TELEMETRY.TELEMETRY.TASK_METRIC_BASELINE b
        MATCH_CONDITION (lm.OBSERVED_TIMESTAMP >= b.CALCULATED_TIMESTAMP)
        ON  lm.TASK_NAME   = b.TASK_NAME
        AND lm.METRIC_NAME = b.METRIC_NAME
        AND lm.SEGMENT_KEY = b.SEGMENT_KEY
  ),
  sev AS (
      SELECT s.*,
             DEV_TELEMETRY.TELEMETRY.FN_BREACH_SEVERITY(
                 IFF(s.LIMIT_TYPE = 'ZSCORE', s.Z_SCORE, s.METRIC_VALUE),
                 s.WARNING_LIMIT, s.CRITICAL_LIMIT, s.LIMIT_TYPE, s.BREACH_DIRECTION) AS SEVERITY
      FROM scored s
      WHERE s.LIMIT_TYPE <> 'ZSCORE' OR s.BASELINE_ID IS NOT NULL   -- cold start: no stats yet
  )
  SELECT TASK_RUN_ID, TASK_NAME, METRIC_NAME, DIMENSION_KEY,
         CASE
           WHEN STARTSWITH(METRIC_NAME, 'DQ_')       THEN 'DATA_QUALITY'
           WHEN METRIC_NAME = 'DURATION_SECONDS'     THEN 'LONG_RUNNING'
           WHEN METRIC_NAME ILIKE '%ROWS%' THEN
                IFF((LIMIT_TYPE = 'ZSCORE' AND Z_SCORE < 0) OR (LIMIT_TYPE <> 'ZSCORE' AND BREACH_DIRECTION = 'BELOW'),
                    'LOW_VOLUME', 'HIGH_VOLUME')
           ELSE 'METRIC_BREACH'
         END,
         SEVERITY,
         METRIC_VALUE,
         IFF(LIMIT_TYPE = 'ZSCORE', MEAN_VALUE, IFF(SEVERITY = 'CRITICAL', CRITICAL_LIMIT, WARNING_LIMIT)),
         ROUND(Z_SCORE, 4),
         BASELINE_ID,
         LEFT('Observed ' || TO_VARCHAR(ROUND(METRIC_VALUE, 2)) ||
              IFF(LIMIT_TYPE = 'ZSCORE',
                  ' vs mean ' || TO_VARCHAR(ROUND(MEAN_VALUE, 2)) || ' (z=' || TO_VARCHAR(ROUND(Z_SCORE, 2)) ||
                  ', baseline ' || TO_VARCHAR(BASELINE_ID) || ')',
                  ' vs limit ' || TO_VARCHAR(IFF(SEVERITY = 'CRITICAL', CRITICAL_LIMIT, WARNING_LIMIT))) ||
              ' at ' || TO_VARCHAR(OBSERVED_TIMESTAMP, 'YYYY-MM-DD HH24:MI') ||
              ' [' || METRIC_NAME || '/' || COALESCE(DIMENSION_KEY, '') || ']', 4000)
  FROM sev
  WHERE SEVERITY IS NOT NULL;

  SELECT COUNT(*) INTO :v_cand FROM DEV_TELEMETRY.TELEMETRY.TMP_EVA_CANDIDATES;

  -- ---------- one candidate per dedup key (highest severity wins) ----------
  CREATE OR REPLACE TEMPORARY TABLE DEV_TELEMETRY.TELEMETRY.TMP_EVA_FINAL AS
  SELECT *
  FROM (
      SELECT c.*,
             c.TASK_NAME || '|' || c.ALERT_TYPE || '|' || COALESCE(c.METRIC_NAME, '') || '|' ||
             COALESCE(c.DIMENSION_KEY, '') AS DEDUP_KEY
      FROM DEV_TELEMETRY.TELEMETRY.TMP_EVA_CANDIDATES c
  ) d
  QUALIFY ROW_NUMBER() OVER (PARTITION BY d.DEDUP_KEY
                             ORDER BY IFF(d.SEVERITY = 'CRITICAL', 1, 2),
                                      ABS(COALESCE(d.Z_SCORE, 0)) DESC,
                                      d.OBSERVED_VALUE DESC NULLS LAST) = 1;

  -- ---------- alert lifecycle (atomic) ----------
  BEGIN TRANSACTION;

  -- escalate: WARNING still open and condition is now CRITICAL
  UPDATE DEV_TELEMETRY.TELEMETRY.ALERT_HISTORY a
     SET STATUS = 'ESCALATED', RESOLVED_TIMESTAMP = :v_ts
    FROM DEV_TELEMETRY.TELEMETRY.TMP_EVA_FINAL f
   WHERE a.DEDUP_KEY = f.DEDUP_KEY
     AND a.STATUS = 'OPEN'
     AND a.SEVERITY = 'WARNING'
     AND f.SEVERITY = 'CRITICAL';
  v_escalated := SQLROWCOUNT;

  -- open: insert once per dedup key
  INSERT INTO DEV_TELEMETRY.TELEMETRY.ALERT_HISTORY (
      TASK_RUN_ID, TASK_NAME, METRIC_NAME, ALERT_TYPE, SEVERITY, OBSERVED_VALUE, EXPECTED_VALUE,
      Z_SCORE, DETECTED_TIMESTAMP, STATUS, DEDUP_KEY, DIMENSION_KEY, BASELINE_ID, ALERT_MESSAGE, ALERT_ROUTE)
  SELECT f.TASK_RUN_ID, f.TASK_NAME, f.METRIC_NAME, f.ALERT_TYPE, f.SEVERITY, f.OBSERVED_VALUE, f.EXPECTED_VALUE,
         f.Z_SCORE, :v_ts, 'OPEN', f.DEDUP_KEY, f.DIMENSION_KEY, f.BASELINE_ID, f.ALERT_MESSAGE, c.ALERT_ROUTE
  FROM DEV_TELEMETRY.TELEMETRY.TMP_EVA_FINAL f
  LEFT JOIN DEV_TELEMETRY.TELEMETRY.TASK_MONITOR_CONFIG c
         ON c.TASK_NAME = f.TASK_NAME AND c.METRIC_NAME = f.METRIC_NAME
  WHERE NOT EXISTS (SELECT 1
                    FROM DEV_TELEMETRY.TELEMETRY.ALERT_HISTORY a
                    WHERE a.DEDUP_KEY = f.DEDUP_KEY AND a.STATUS = 'OPEN');
  v_opened := SQLROWCOUNT;

  -- resolve: open alerts in this schema whose condition no longer holds
  UPDATE DEV_TELEMETRY.TELEMETRY.ALERT_HISTORY a
     SET STATUS = 'RESOLVED', RESOLVED_TIMESTAMP = :v_ts
   WHERE a.STATUS = 'OPEN'
     AND STARTSWITH(a.TASK_NAME, :v_prefix)
     AND NOT EXISTS (SELECT 1 FROM DEV_TELEMETRY.TELEMETRY.TMP_EVA_FINAL f WHERE f.DEDUP_KEY = a.DEDUP_KEY);
  v_resolved := SQLROWCOUNT;

  COMMIT;

  RETURN 'candidates=' || v_cand || ', opened=' || v_opened ||
         ', escalated=' || v_escalated || ', resolved=' || v_resolved;
EXCEPTION
  WHEN OTHER THEN
    ROLLBACK;
    RAISE;
END;
$$;


/* =====================================================================================
   9. NOTIFICATIONS  -  SP_NOTIFY_ALERTS
   -------------------------------------------------------------------------------------
   One summary email per cycle for OPEN alerts not yet notified, where ALERT_ENABLED.
   NOTIFIED_TIMESTAMP is set only after the email call succeeds (at-least-once delivery).
   ServiceNow: point <ALERT_RECIPIENTS> at the ServiceNow inbound address (if it can be a
   verified Snowflake user email) or switch to a webhook integration later.
   ===================================================================================== */
CREATE OR REPLACE PROCEDURE DEV_TELEMETRY.TELEMETRY.SP_NOTIFY_ALERTS(
    P_INTEGRATION VARCHAR,
    P_RECIPIENTS  VARCHAR,
    P_ENV         VARCHAR DEFAULT 'DEV')
RETURNS VARCHAR
LANGUAGE SQL
EXECUTE AS CALLER
AS
$$
DECLARE
  v_ts      TIMESTAMP_LTZ;
  v_cnt     NUMBER DEFAULT 0;
  v_crit    NUMBER DEFAULT 0;
  v_warn    NUMBER DEFAULT 0;
  v_lines   VARCHAR;
  v_subject VARCHAR;
  v_body    VARCHAR;
BEGIN
  v_ts := CURRENT_TIMESTAMP();

  CREATE OR REPLACE TEMPORARY TABLE DEV_TELEMETRY.TELEMETRY.TMP_NTF AS
    SELECT a.ALERT_ID, a.SEVERITY, a.ALERT_TYPE, a.TASK_NAME, a.METRIC_NAME,
           a.ALERT_MESSAGE, a.DETECTED_TIMESTAMP,
           IFF(a.SEVERITY = 'CRITICAL', 1, 2) AS SEV_ORDER
    FROM DEV_TELEMETRY.TELEMETRY.ALERT_HISTORY a
    JOIN DEV_TELEMETRY.TELEMETRY.TASK_MONITOR_CONFIG c
      ON c.TASK_NAME = a.TASK_NAME AND c.METRIC_NAME = a.METRIC_NAME
    WHERE a.STATUS = 'OPEN'
      AND a.NOTIFIED_TIMESTAMP IS NULL
      AND c.ALERT_ENABLED;

  SELECT COUNT(*), COUNT_IF(SEVERITY = 'CRITICAL'), COUNT_IF(SEVERITY = 'WARNING')
    INTO :v_cnt, :v_crit, :v_warn
    FROM DEV_TELEMETRY.TELEMETRY.TMP_NTF;

  IF (v_cnt = 0) THEN
    RETURN 'no new alerts';
  END IF;

  SELECT LISTAGG(SEVERITY || ' | ' || ALERT_TYPE || ' | ' || TASK_NAME || CHR(10) ||
                 '    ' || COALESCE(ALERT_MESSAGE, ''), CHR(10))
           WITHIN GROUP (ORDER BY SEV_ORDER, TASK_NAME)
    INTO :v_lines
    FROM (SELECT * FROM DEV_TELEMETRY.TELEMETRY.TMP_NTF
          ORDER BY SEV_ORDER, DETECTED_TIMESTAMP
          LIMIT 100);

  v_subject := '[' || P_ENV || '] Virtuous telemetry: ' || v_crit || ' critical, ' || v_warn || ' warning';
  v_body    := 'New alerts detected ' || TO_VARCHAR(v_ts) || CHR(10) || CHR(10) ||
               v_lines || CHR(10) || CHR(10) ||
               'Total new alerts: ' || v_cnt || IFF(v_cnt > 100, ' (first 100 listed)', '') || CHR(10) ||
               'Detail: SELECT * FROM DEV_TELEMETRY.TELEMETRY.VW_TASK_CURRENT_HEALTH;';

  CALL SYSTEM$SEND_EMAIL(:P_INTEGRATION, :P_RECIPIENTS, :v_subject, :v_body);

  UPDATE DEV_TELEMETRY.TELEMETRY.ALERT_HISTORY
     SET NOTIFIED_TIMESTAMP = :v_ts
   WHERE ALERT_ID IN (SELECT ALERT_ID FROM DEV_TELEMETRY.TELEMETRY.TMP_NTF);

  RETURN 'notified ' || v_cnt || ' alert(s)';
END;
$$;


/* =====================================================================================
   10. OPERATIONAL VIEWS
   ===================================================================================== */

-- 10.1 Latest execution per task: status, duration, row counts, errors
CREATE OR REPLACE VIEW DEV_TELEMETRY.TELEMETRY.VW_TASK_LATEST_RUN AS
WITH h AS (
    SELECT *
    FROM DEV_TELEMETRY.TELEMETRY.TASK_EXECUTION_HISTORY
    WHERE SCHEDULED_TIME >= DATEADD('day', -35, CURRENT_TIMESTAMP())
),
agg AS (
    SELECT TASK_NAME,
           MAX(COALESCE(END_TIME, SCHEDULED_TIME))         AS LAST_CHECKED_TIME,   -- incl. SKIPPED
           MAX(IFF(STATE = 'SUCCEEDED', END_TIME, NULL))   AS LAST_SUCCESS_TIME,
           COUNT_IF(STATE IN ('FAILED', 'FAILED_AND_AUTO_SUSPENDED')
                    AND SCHEDULED_TIME >= DATEADD('hour', -24, CURRENT_TIMESTAMP())) AS FAILURES_24H,
           COUNT_IF(STATE = 'SUCCEEDED'
                    AND SCHEDULED_TIME >= DATEADD('hour', -24, CURRENT_TIMESTAMP())) AS SUCCESSES_24H
    FROM h
    GROUP BY TASK_NAME
),
lr AS (   -- latest run that actually executed
    SELECT *
    FROM h
    WHERE STATE <> 'SKIPPED'
    QUALIFY ROW_NUMBER() OVER (PARTITION BY TASK_NAME
                               ORDER BY SCHEDULED_TIME DESC, END_TIME DESC NULLS LAST) = 1
)
SELECT a.TASK_NAME,
       lr.GRAPH_NAME,
       lr.TASK_RUN_ID,
       lr.STATE,
       lr.ERROR_CODE,
       lr.ERROR_MESSAGE,
       lr.SCHEDULED_TIME,
       lr.START_TIME,
       lr.END_TIME,
       lr.DURATION_SECONDS,
       lr.ROWS_INSERTED,
       lr.ROWS_UPDATED,
       lr.ROWS_DELETED,
       lr.ROWS_INSERTED + lr.ROWS_UPDATED + lr.ROWS_DELETED AS ROWS_TOTAL,
       lr.SOURCE_NAME,
       lr.STREAM_NAME,
       lr.TARGET_NAME,
       a.LAST_CHECKED_TIME,
       a.LAST_SUCCESS_TIME,
       a.FAILURES_24H,
       a.SUCCESSES_24H
FROM agg a
LEFT JOIN lr ON lr.TASK_NAME = a.TASK_NAME;

-- 10.2 Current health per monitored task
--      Precedence: SUSPENDED > FAILED > OVERDUE > LONG_RUNNING > NO_RUNS_YET > HEALTHY.
--      RUNNING is transient and only exists live in TASK_HISTORY (see query 14.1); if it
--      must appear here, the evaluator can write a small live-state table (ask Michael).
CREATE OR REPLACE VIEW DEV_TELEMETRY.TELEMETRY.VW_TASK_CURRENT_HEALTH AS
WITH monitored AS (
    SELECT TASK_NAME, MAX(EXPECTED_FREQUENCY_MINUTES) AS EXPECTED_FREQUENCY_MINUTES
    FROM DEV_TELEMETRY.TELEMETRY.TASK_MONITOR_CONFIG
    WHERE MONITOR_ENABLED
    GROUP BY TASK_NAME
),
open_alerts AS (
    SELECT TASK_NAME,
           COUNT(*)                                     AS OPEN_ALERT_COUNT,
           MAX(IFF(SEVERITY = 'CRITICAL', 2, 1))        AS MAX_SEV,
           MAX(IFF(ALERT_TYPE = 'SUSPENDED', 1, 0))     AS F_SUSPENDED,
           MAX(IFF(ALERT_TYPE = 'FAILED', 1, 0))        AS F_FAILED,
           MAX(IFF(ALERT_TYPE = 'OVERDUE', 1, 0))       AS F_OVERDUE,
           MAX(IFF(ALERT_TYPE = 'LONG_RUNNING', 1, 0))  AS F_LONG_RUNNING,
           LISTAGG(DISTINCT ALERT_TYPE, ', ') WITHIN GROUP (ORDER BY ALERT_TYPE) AS OPEN_ALERT_TYPES,
           MIN(DETECTED_TIMESTAMP)                      AS OLDEST_OPEN_ALERT
    FROM DEV_TELEMETRY.TELEMETRY.ALERT_HISTORY
    WHERE STATUS = 'OPEN'
    GROUP BY TASK_NAME
)
SELECT m.TASK_NAME,
       CASE WHEN oa.F_SUSPENDED    = 1 THEN 'SUSPENDED'
            WHEN oa.F_FAILED       = 1 THEN 'FAILED'
            WHEN oa.F_OVERDUE      = 1 THEN 'OVERDUE'
            WHEN oa.F_LONG_RUNNING = 1 THEN 'LONG_RUNNING'
            WHEN lr.TASK_NAME IS NULL  THEN 'NO_RUNS_YET'
            ELSE 'HEALTHY'
       END                                              AS HEALTH_STATUS,
       COALESCE(oa.OPEN_ALERT_COUNT, 0)                 AS OPEN_ALERT_COUNT,
       DECODE(oa.MAX_SEV, 2, 'CRITICAL', 1, 'WARNING')  AS MAX_OPEN_SEVERITY,
       oa.OPEN_ALERT_TYPES,
       oa.OLDEST_OPEN_ALERT,
       m.EXPECTED_FREQUENCY_MINUTES,
       lr.STATE                                         AS LAST_RUN_STATE,
       lr.END_TIME                                      AS LAST_RUN_END_TIME,
       lr.LAST_SUCCESS_TIME,
       lr.LAST_CHECKED_TIME,
       lr.DURATION_SECONDS                              AS LAST_DURATION_SECONDS,
       lr.ROWS_TOTAL                                    AS LAST_ROWS_TOTAL,
       lr.FAILURES_24H,
       lr.ERROR_MESSAGE                                 AS LAST_ERROR_MESSAGE
FROM monitored m
LEFT JOIN DEV_TELEMETRY.TELEMETRY.VW_TASK_LATEST_RUN lr ON lr.TASK_NAME = m.TASK_NAME
LEFT JOIN open_alerts oa                                ON oa.TASK_NAME = m.TASK_NAME;

-- 10.3 Metric anomaly: observed value vs the baseline active at observation time
CREATE OR REPLACE VIEW DEV_TELEMETRY.TELEMETRY.VW_TASK_METRIC_ANOMALY AS
WITH obs AS (
    SELECT m.METRIC_HISTORY_ID, m.TASK_RUN_ID, m.TASK_NAME, m.METRIC_NAME, m.DIMENSION_KEY,
           m.METRIC_VALUE, m.OBSERVED_TIMESTAMP,
           IFF(c.DAY_OF_WEEK_SEGMENT, 'DOW_' || DAYOFWEEKISO(m.OBSERVED_TIMESTAMP), 'ALL') AS SEGMENT_KEY,
           c.WARNING_LIMIT, c.CRITICAL_LIMIT, c.BREACH_DIRECTION
    FROM DEV_TELEMETRY.TELEMETRY.TASK_METRIC_HISTORY m
    JOIN DEV_TELEMETRY.TELEMETRY.TASK_MONITOR_CONFIG c
      ON c.TASK_NAME = m.TASK_NAME AND c.METRIC_NAME = m.METRIC_NAME
    WHERE c.LIMIT_TYPE = 'ZSCORE'
      AND m.OBSERVED_TIMESTAMP >= DATEADD('day', -30, CURRENT_TIMESTAMP())
)
SELECT o.TASK_NAME,
       o.METRIC_NAME,
       o.DIMENSION_KEY,
       o.TASK_RUN_ID,
       o.OBSERVED_TIMESTAMP,
       o.METRIC_VALUE                                                      AS OBSERVED_VALUE,
       o.SEGMENT_KEY,
       b.BASELINE_ID,
       b.SAMPLE_COUNT,
       b.MEAN_VALUE,
       b.STDDEV_VALUE,
       b.LOWER_LIMIT,
       b.UPPER_LIMIT,
       ROUND((o.METRIC_VALUE - b.MEAN_VALUE) / NULLIF(b.STDDEV_VALUE, 0), 4) AS Z_SCORE,
       COALESCE(
           DEV_TELEMETRY.TELEMETRY.FN_BREACH_SEVERITY(
               (o.METRIC_VALUE - b.MEAN_VALUE) / NULLIF(b.STDDEV_VALUE, 0),
               o.WARNING_LIMIT, o.CRITICAL_LIMIT, 'ZSCORE', o.BREACH_DIRECTION),
           IFF(b.BASELINE_ID IS NULL, 'NO_BASELINE', 'NORMAL'))           AS SEVERITY
FROM obs o
ASOF JOIN DEV_TELEMETRY.TELEMETRY.TASK_METRIC_BASELINE b
  MATCH_CONDITION (o.OBSERVED_TIMESTAMP >= b.CALCULATED_TIMESTAMP)
  ON  o.TASK_NAME   = b.TASK_NAME
  AND o.METRIC_NAME = b.METRIC_NAME
  AND o.SEGMENT_KEY = b.SEGMENT_KEY;


/* =====================================================================================
   11. TELEMETRY TASKS + INDEPENDENT WATCHDOG
   -------------------------------------------------------------------------------------
   Cost: offset 5 min after the loads (:00/:15/:30/:45) so the work lands while the load
   warehouse is often still warm. Serverless alternative for short runs: replace
   WAREHOUSE = ... with USER_TASK_MANAGED_INITIAL_WAREHOUSE_SIZE = 'XSMALL'
   (needs EXECUTE MANAGED TASK).
   ===================================================================================== */
CREATE OR REPLACE TASK DEV_TELEMETRY.TELEMETRY.TSK_TELEMETRY_COLLECT
  WAREHOUSE = <TELEMETRY_WH>
  SCHEDULE = 'USING CRON 5,20,35,50 * * * * America/New_York'
  ALLOW_OVERLAPPING_EXECUTION = FALSE
  USER_TASK_TIMEOUT_MS = 600000
  TIMEZONE = 'America/New_York'
  COMMENT = 'Append task execution + metric facts (landing schema and telemetry schema)'
AS
EXECUTE IMMEDIATE $$
BEGIN
  CALL DEV_TELEMETRY.TELEMETRY.SP_COLLECT_TASK_TELEMETRY('<LANDING_DB>', '<LANDING_SCHEMA>', 6);
  CALL DEV_TELEMETRY.TELEMETRY.SP_COLLECT_TASK_TELEMETRY('DEV_TELEMETRY', 'TELEMETRY', 6);
END;
$$;

CREATE OR REPLACE TASK DEV_TELEMETRY.TELEMETRY.TSK_TELEMETRY_EVALUATE
  WAREHOUSE = <TELEMETRY_WH>
  USER_TASK_TIMEOUT_MS = 600000
  TIMEZONE = 'America/New_York'
  COMMENT = 'Evaluate rules, open/resolve alerts, notify'
  AFTER DEV_TELEMETRY.TELEMETRY.TSK_TELEMETRY_COLLECT
AS
EXECUTE IMMEDIATE $$
BEGIN
  CALL DEV_TELEMETRY.TELEMETRY.SP_EVALUATE_TASK_HEALTH('<LANDING_DB>', '<LANDING_SCHEMA>');
  CALL DEV_TELEMETRY.TELEMETRY.SP_EVALUATE_TASK_HEALTH('DEV_TELEMETRY', 'TELEMETRY');
  CALL DEV_TELEMETRY.TELEMETRY.SP_NOTIFY_ALERTS('<EMAIL_INTEGRATION>', '<ALERT_RECIPIENTS>', 'DEV');
END;
$$;

CREATE OR REPLACE TASK DEV_TELEMETRY.TELEMETRY.TSK_TELEMETRY_BASELINE
  WAREHOUSE = <TELEMETRY_WH>
  SCHEDULE = 'USING CRON 17 2 * * * America/New_York'
  USER_TASK_TIMEOUT_MS = 1800000
  TIMEZONE = 'America/New_York'
  COMMENT = 'Daily: reconciliation DQ metrics, daily rollups, new baseline versions'
AS
EXECUTE IMMEDIATE $$
BEGIN
  CALL DEV_TELEMETRY.TELEMETRY.SP_COLLECT_DQ_RECON('<LANDING_DB>', '<LANDING_SCHEMA>');
  CALL DEV_TELEMETRY.TELEMETRY.SP_CALC_TASK_BASELINE();
END;
$$;

-- Watchdog: independent of the task graph. Fires if EVALUATE has not succeeded in 45 min
-- (proves collect + evaluate both ran). Re-emails every 30 min while broken - intended.
CREATE OR REPLACE ALERT DEV_TELEMETRY.TELEMETRY.ALR_TELEMETRY_WATCHDOG
  WAREHOUSE = <TELEMETRY_WH>
  SCHEDULE = '30 MINUTE'
  IF (EXISTS (
        SELECT 1
        FROM (SELECT MAX(COMPLETED_TIME) AS LAST_OK
              FROM TABLE(DEV_TELEMETRY.INFORMATION_SCHEMA.TASK_HISTORY(
                     SCHEDULED_TIME_RANGE_START => DATEADD('hour', -6, CURRENT_TIMESTAMP()),
                     TASK_NAME => 'TSK_TELEMETRY_EVALUATE'))
              WHERE STATE = 'SUCCEEDED' AND SCHEMA_NAME = 'TELEMETRY')
        WHERE LAST_OK IS NULL
           OR LAST_OK < DATEADD('minute', -45, CURRENT_TIMESTAMP())))
  THEN
    CALL SYSTEM$SEND_EMAIL(
      '<EMAIL_INTEGRATION>', '<ALERT_RECIPIENTS>',
      '[DEV] CRITICAL - Virtuous telemetry is not running',
      'TSK_TELEMETRY_EVALUATE has not succeeded in 45+ minutes. Pipeline failures are NOT being detected. Check: SHOW TASKS IN SCHEMA DEV_TELEMETRY.TELEMETRY; and INFORMATION_SCHEMA.TASK_HISTORY.');

-- Resume LAST (after the test kit passes): enables the root and all dependents
-- SELECT SYSTEM$TASK_DEPENDENTS_ENABLE('DEV_TELEMETRY.TELEMETRY.TSK_TELEMETRY_COLLECT');
-- ALTER TASK  DEV_TELEMETRY.TELEMETRY.TSK_TELEMETRY_BASELINE RESUME;
-- ALTER ALERT DEV_TELEMETRY.TELEMETRY.ALR_TELEMETRY_WATCHDOG RESUME;


/* =====================================================================================
   12. CONFIG SEED  (re-runnable: inserts missing rows only, never overwrites tuned values)
   -------------------------------------------------------------------------------------
   Statistical rules start in SHADOW MODE (ALERT_ENABLED = FALSE): alerts are recorded
   and visible in the views, but nobody is paged until the baseline has matured and the
   false-positive rate has been reviewed. Deterministic rules alert from day one.
   NOTE: with DAY_OF_WEEK_SEGMENT, 180 days gives only ~25 samples per weekday, so a
   MIN_SAMPLE_COUNT of 30 would never baseline - DAILY_ROWS_TOTAL uses 8.
   ===================================================================================== */
SHOW TASKS LIKE 'TSK_LOAD_VIRTUOUS_%' IN SCHEMA <LANDING_DB>.<LANDING_SCHEMA>;

CREATE OR REPLACE TEMPORARY TABLE DEV_TELEMETRY.TELEMETRY.TMP_SEED_TASKS AS
SELECT UPPER("database_name" || '.' || "schema_name" || '.' || "name") AS TASK_NAME,
       "schedule"::VARCHAR                                              AS SCHEDULE,
       CASE WHEN "schedule"::VARCHAR ILIKE '% MINUTE%'
            THEN TRY_TO_NUMBER(SPLIT_PART(TRIM("schedule"::VARCHAR), ' ', 1)) END AS FREQ_MIN
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));

SELECT COUNT(*) AS LOAD_TASKS_FOUND FROM DEV_TELEMETRY.TELEMETRY.TMP_SEED_TASKS;   -- expect 36

MERGE INTO DEV_TELEMETRY.TELEMETRY.TASK_MONITOR_CONFIG t
USING (
    SELECT s.TASK_NAME,
           r.METRIC_NAME,
           IFF(r.METRIC_NAME = 'SCHEDULE', COALESCE(s.FREQ_MIN, 15), NULL) AS EXPECTED_FREQUENCY_MINUTES,
           r.WARNING_LIMIT, r.CRITICAL_LIMIT, r.LIMIT_TYPE, r.BREACH_DIRECTION,
           r.BASELINE_WINDOW_DAYS, r.MIN_SAMPLE_COUNT, r.DAY_OF_WEEK_SEGMENT, r.ALERT_ENABLED
    FROM DEV_TELEMETRY.TELEMETRY.TMP_SEED_TASKS s
    CROSS JOIN (
        SELECT COLUMN1 AS METRIC_NAME, COLUMN2 AS WARNING_LIMIT, COLUMN3 AS CRITICAL_LIMIT,
               COLUMN4 AS LIMIT_TYPE, COLUMN5 AS BREACH_DIRECTION, COLUMN6 AS BASELINE_WINDOW_DAYS,
               COLUMN7 AS MIN_SAMPLE_COUNT, COLUMN8 AS DAY_OF_WEEK_SEGMENT, COLUMN9 AS ALERT_ENABLED
        FROM VALUES
          ('TASK_STATE',            NULL::NUMBER, NULL::NUMBER, 'ABSOLUTE', 'ABOVE', 180, 30, FALSE, TRUE ),
          ('SCHEDULE',              10,   30,   'ABSOLUTE', 'ABOVE', 180, 30, FALSE, TRUE ),  -- tolerance minutes
          ('DURATION_SECONDS',      600,  1800, 'ABSOLUTE', 'ABOVE', 180, 30, FALSE, TRUE ),  -- seconds
          ('ROWS_TOTAL',            2,    3,    'ZSCORE',   'BOTH',  180, 30, FALSE, FALSE),  -- shadow
          ('DAILY_ROWS_TOTAL',      2,    3,    'ZSCORE',   'BOTH',  180, 8,  TRUE,  FALSE),  -- shadow
          ('STREAM_HOURS_TO_STALE', 48,   12,   'ABSOLUTE', 'BELOW', 180, 30, FALSE, TRUE ),  -- hours
          ('DQ_RECON_DIFF',         0,    10,   'ABSOLUTE', 'ABOVE', 180, 30, FALSE, TRUE ),  -- rows
          ('DQ_RECON_CHECK_ERROR',  0,    NULL, 'ABSOLUTE', 'ABOVE', 180, 30, FALSE, TRUE )
    ) r
) src
ON  t.TASK_NAME   = src.TASK_NAME
AND t.METRIC_NAME = src.METRIC_NAME
WHEN NOT MATCHED THEN INSERT (
    TASK_NAME, METRIC_NAME, EXPECTED_FREQUENCY_MINUTES, WARNING_LIMIT, CRITICAL_LIMIT,
    BASELINE_WINDOW_DAYS, MIN_SAMPLE_COUNT, DAY_OF_WEEK_SEGMENT, MONITOR_ENABLED, ALERT_ENABLED,
    LIMIT_TYPE, BREACH_DIRECTION, ALERT_ROUTE, UPDATED_BY, UPDATED_TIMESTAMP)
VALUES (
    src.TASK_NAME, src.METRIC_NAME, src.EXPECTED_FREQUENCY_MINUTES, src.WARNING_LIMIT, src.CRITICAL_LIMIT,
    src.BASELINE_WINDOW_DAYS, src.MIN_SAMPLE_COUNT, src.DAY_OF_WEEK_SEGMENT, TRUE, src.ALERT_ENABLED,
    src.LIMIT_TYPE, src.BREACH_DIRECTION, 'EMAIL:DATA_SUPPORT', CURRENT_USER(), CURRENT_TIMESTAMP());

-- Telemetry monitors itself for failures (the watchdog covers "not running at all")
MERGE INTO DEV_TELEMETRY.TELEMETRY.TASK_MONITOR_CONFIG t
USING (
    SELECT COLUMN1 AS TASK_NAME, COLUMN2 AS METRIC_NAME, COLUMN3 AS FREQ,
           COLUMN4 AS WARNING_LIMIT, COLUMN5 AS CRITICAL_LIMIT
    FROM VALUES
      ('DEV_TELEMETRY.TELEMETRY.TSK_TELEMETRY_COLLECT',  'TASK_STATE', NULL::NUMBER, NULL::NUMBER, NULL::NUMBER),
      ('DEV_TELEMETRY.TELEMETRY.TSK_TELEMETRY_EVALUATE', 'TASK_STATE', NULL, NULL, NULL),
      ('DEV_TELEMETRY.TELEMETRY.TSK_TELEMETRY_BASELINE', 'TASK_STATE', NULL, NULL, NULL),
      ('DEV_TELEMETRY.TELEMETRY.TSK_TELEMETRY_BASELINE', 'SCHEDULE',   1440, 60,   180)
) src
ON t.TASK_NAME = src.TASK_NAME AND t.METRIC_NAME = src.METRIC_NAME
WHEN NOT MATCHED THEN INSERT (
    TASK_NAME, METRIC_NAME, EXPECTED_FREQUENCY_MINUTES, WARNING_LIMIT, CRITICAL_LIMIT,
    LIMIT_TYPE, BREACH_DIRECTION, MONITOR_ENABLED, ALERT_ENABLED, ALERT_ROUTE, UPDATED_BY, UPDATED_TIMESTAMP)
VALUES (
    src.TASK_NAME, src.METRIC_NAME, src.FREQ, src.WARNING_LIMIT, src.CRITICAL_LIMIT,
    'ABSOLUTE', 'ABOVE', TRUE, TRUE, 'EMAIL:DATA_SUPPORT', CURRENT_USER(), CURRENT_TIMESTAMP());


/* =====================================================================================
   13. US-3.3 TEST KIT  -  "Inject known failures" + "Validate status, exceptions, evidence"
   -------------------------------------------------------------------------------------
   Expected after phase B:
     TSK_TEST_FAIL       FAILED        CRITICAL
     TSK_TEST_SLOW       LONG_RUNNING  WARNING live (run evaluate during the wait), CRITICAL after collect (180 s > 120)
     TSK_TEST_VOLUME     LOW_VOLUME    CRITICAL (0 rows vs synthetic mean 100, sd 10 -> z = -10)
     TSK_TEST_SUSPENDED  SUSPENDED     CRITICAL
     TSK_TEST_OVERDUE    OVERDUE       CRITICAL (expected every 5 min, never ran)
     TSK_TEST_DQ         DATA_QUALITY  CRITICAL (DQ_NULL_PCT 12.5 > 5)
   Phase C: switch volume to 1000 rows -> HIGH_VOLUME opens, LOW_VOLUME resolves.
   Phase D: fix everything -> every test alert RESOLVED. Screenshot 13.6 for the story.
   ===================================================================================== */

-- 13.1 Test tasks (created SUSPENDED; EXECUTE TASK runs a suspended task once)
CREATE OR REPLACE TASK DEV_TELEMETRY.TELEMETRY_TEST.TSK_TEST_FAIL
  WAREHOUSE = <TELEMETRY_WH> SCHEDULE = '11520 MINUTE'
AS SELECT 1/0;

CREATE OR REPLACE TASK DEV_TELEMETRY.TELEMETRY_TEST.TSK_TEST_SLOW
  WAREHOUSE = <TELEMETRY_WH> SCHEDULE = '11520 MINUTE'
AS SELECT SYSTEM$WAIT(3, 'MINUTES');

CREATE OR REPLACE TASK DEV_TELEMETRY.TELEMETRY_TEST.TSK_TEST_VOLUME
  WAREHOUSE = <TELEMETRY_WH> SCHEDULE = '11520 MINUTE'
AS CALL SYSTEM$SET_RETURN_VALUE('{"inserted":0,"updated":0,"deleted":0}');

CREATE OR REPLACE TASK DEV_TELEMETRY.TELEMETRY_TEST.TSK_TEST_SUSPENDED
  WAREHOUSE = <TELEMETRY_WH> SCHEDULE = '60 MINUTE'
AS SELECT 1;

CREATE OR REPLACE TASK DEV_TELEMETRY.TELEMETRY_TEST.TSK_TEST_OVERDUE
  WAREHOUSE = <TELEMETRY_WH> SCHEDULE = '60 MINUTE'    -- resumed below; first run only after 60 min
AS SELECT 1;

-- 13.2 Test config (UPDATED_TIMESTAMP backdated so OVERDUE does not wait for the grace period)
MERGE INTO DEV_TELEMETRY.TELEMETRY.TASK_MONITOR_CONFIG t
USING (
    SELECT COLUMN1 AS TASK_NAME, COLUMN2 AS METRIC_NAME, COLUMN3 AS FREQ,
           COLUMN4 AS WARNING_LIMIT, COLUMN5 AS CRITICAL_LIMIT, COLUMN6 AS LIMIT_TYPE, COLUMN7 AS BREACH_DIRECTION
    FROM VALUES
      ('DEV_TELEMETRY.TELEMETRY_TEST.TSK_TEST_FAIL',      'TASK_STATE',       NULL::NUMBER, NULL::NUMBER, NULL::NUMBER, 'ABSOLUTE', 'ABOVE'),
      ('DEV_TELEMETRY.TELEMETRY_TEST.TSK_TEST_SLOW',      'DURATION_SECONDS', NULL, 60,   120,  'ABSOLUTE', 'ABOVE'),
      ('DEV_TELEMETRY.TELEMETRY_TEST.TSK_TEST_VOLUME',    'ROWS_TOTAL',       NULL, 2,    3,    'ZSCORE',   'BOTH'),
      ('DEV_TELEMETRY.TELEMETRY_TEST.TSK_TEST_SUSPENDED', 'SCHEDULE',         60,   10,   30,   'ABSOLUTE', 'ABOVE'),
      ('DEV_TELEMETRY.TELEMETRY_TEST.TSK_TEST_OVERDUE',   'SCHEDULE',         5,    1,    2,    'ABSOLUTE', 'ABOVE'),
      ('DEV_TELEMETRY.TELEMETRY_TEST.TSK_TEST_DQ',        'DQ_NULL_PCT',      NULL, 1,    5,    'ABSOLUTE', 'ABOVE')
) src
ON t.TASK_NAME = src.TASK_NAME AND t.METRIC_NAME = src.METRIC_NAME
WHEN NOT MATCHED THEN INSERT (
    TASK_NAME, METRIC_NAME, EXPECTED_FREQUENCY_MINUTES, WARNING_LIMIT, CRITICAL_LIMIT,
    LIMIT_TYPE, BREACH_DIRECTION, MONITOR_ENABLED, ALERT_ENABLED, ALERT_ROUTE, UPDATED_BY, UPDATED_TIMESTAMP)
VALUES (
    src.TASK_NAME, src.METRIC_NAME, src.FREQ, src.WARNING_LIMIT, src.CRITICAL_LIMIT,
    src.LIMIT_TYPE, src.BREACH_DIRECTION, TRUE, TRUE, 'TEST', CURRENT_USER(),
    DATEADD('hour', -1, CURRENT_TIMESTAMP()));

-- 13.3 Synthetic baseline for the volume test (calculated before the run, so ASOF picks it)
INSERT INTO DEV_TELEMETRY.TELEMETRY.TASK_METRIC_BASELINE (
    TASK_NAME, METRIC_NAME, SEGMENT_KEY, WINDOW_START, WINDOW_END, SAMPLE_COUNT,
    MEAN_VALUE, STDDEV_VALUE, LOWER_LIMIT, UPPER_LIMIT, CALCULATED_TIMESTAMP)
SELECT 'DEV_TELEMETRY.TELEMETRY_TEST.TSK_TEST_VOLUME', 'ROWS_TOTAL', 'ALL',
       DATEADD('day', -180, CURRENT_TIMESTAMP()), DATEADD('hour', -1, CURRENT_TIMESTAMP()),
       999, 100, 10, 70, 130, DATEADD('hour', -1, CURRENT_TIMESTAMP());

-- 13.4 Phase B - inject
EXECUTE TASK DEV_TELEMETRY.TELEMETRY_TEST.TSK_TEST_FAIL;
EXECUTE TASK DEV_TELEMETRY.TELEMETRY_TEST.TSK_TEST_SLOW;
EXECUTE TASK DEV_TELEMETRY.TELEMETRY_TEST.TSK_TEST_VOLUME;
ALTER TASK  DEV_TELEMETRY.TELEMETRY_TEST.TSK_TEST_OVERDUE RESUME;   -- TSK_TEST_SUSPENDED stays suspended

INSERT INTO DEV_TELEMETRY.TELEMETRY.TASK_METRIC_HISTORY (
    TASK_RUN_ID, TASK_NAME, METRIC_NAME, METRIC_VALUE, UNIT_OF_MEASURE,
    DIMENSION_KEY, OBSERVED_TIMESTAMP, COLLECTED_TIMESTAMP)
SELECT NULL, 'DEV_TELEMETRY.TELEMETRY_TEST.TSK_TEST_DQ', 'DQ_NULL_PCT', 12.5, 'PCT',
       'COLUMN:EMAIL', CURRENT_TIMESTAMP(), CURRENT_TIMESTAMP();

-- ~1-2 min later (SLOW still running): live LONG_RUNNING check
CALL DEV_TELEMETRY.TELEMETRY.SP_EVALUATE_TASK_HEALTH('DEV_TELEMETRY', 'TELEMETRY_TEST');

-- after ~4 min (SLOW finished):
SELECT NAME, STATE, ERROR_MESSAGE, QUERY_START_TIME, COMPLETED_TIME, RETURN_VALUE
FROM TABLE(DEV_TELEMETRY.INFORMATION_SCHEMA.TASK_HISTORY(
       SCHEDULED_TIME_RANGE_START => DATEADD('hour', -1, CURRENT_TIMESTAMP())))
WHERE SCHEMA_NAME = 'TELEMETRY_TEST'
ORDER BY SCHEDULED_TIME DESC;

CALL DEV_TELEMETRY.TELEMETRY.SP_COLLECT_TASK_TELEMETRY('DEV_TELEMETRY', 'TELEMETRY_TEST', 6);
CALL DEV_TELEMETRY.TELEMETRY.SP_EVALUATE_TASK_HEALTH('DEV_TELEMETRY', 'TELEMETRY_TEST');

-- 13.5 Phase C - high volume
ALTER TASK DEV_TELEMETRY.TELEMETRY_TEST.TSK_TEST_VOLUME
  MODIFY AS CALL SYSTEM$SET_RETURN_VALUE('{"inserted":1000,"updated":0,"deleted":0}');
EXECUTE TASK DEV_TELEMETRY.TELEMETRY_TEST.TSK_TEST_VOLUME;
-- wait ~1 min, then collect + evaluate (same two CALLs as above)

-- Phase D - fix everything, then collect + evaluate again -> all RESOLVED
ALTER TASK DEV_TELEMETRY.TELEMETRY_TEST.TSK_TEST_FAIL   MODIFY AS SELECT 1;
ALTER TASK DEV_TELEMETRY.TELEMETRY_TEST.TSK_TEST_SLOW   MODIFY AS SELECT 1;
ALTER TASK DEV_TELEMETRY.TELEMETRY_TEST.TSK_TEST_VOLUME
  MODIFY AS CALL SYSTEM$SET_RETURN_VALUE('{"inserted":100,"updated":0,"deleted":0}');
EXECUTE TASK DEV_TELEMETRY.TELEMETRY_TEST.TSK_TEST_FAIL;
EXECUTE TASK DEV_TELEMETRY.TELEMETRY_TEST.TSK_TEST_SLOW;
EXECUTE TASK DEV_TELEMETRY.TELEMETRY_TEST.TSK_TEST_VOLUME;
ALTER TASK   DEV_TELEMETRY.TELEMETRY_TEST.TSK_TEST_SUSPENDED RESUME;
EXECUTE TASK DEV_TELEMETRY.TELEMETRY_TEST.TSK_TEST_SUSPENDED;   -- gives it a run, so no OVERDUE follows
EXECUTE TASK DEV_TELEMETRY.TELEMETRY_TEST.TSK_TEST_OVERDUE;
INSERT INTO DEV_TELEMETRY.TELEMETRY.TASK_METRIC_HISTORY (
    TASK_RUN_ID, TASK_NAME, METRIC_NAME, METRIC_VALUE, UNIT_OF_MEASURE,
    DIMENSION_KEY, OBSERVED_TIMESTAMP, COLLECTED_TIMESTAMP)
SELECT NULL, 'DEV_TELEMETRY.TELEMETRY_TEST.TSK_TEST_DQ', 'DQ_NULL_PCT', 0, 'PCT',
       'COLUMN:EMAIL', CURRENT_TIMESTAMP(), CURRENT_TIMESTAMP();
-- wait ~1 min, then collect + evaluate

-- 13.6 Evidence (attach to US-3.3)
SELECT ALERT_TYPE, SEVERITY, TASK_NAME, STATUS, DETECTED_TIMESTAMP, RESOLVED_TIMESTAMP,
       DATEDIFF('minute', DETECTED_TIMESTAMP, RESOLVED_TIMESTAMP) AS MINUTES_OPEN,
       OBSERVED_VALUE, EXPECTED_VALUE, Z_SCORE, BASELINE_ID, ALERT_MESSAGE
FROM DEV_TELEMETRY.TELEMETRY.ALERT_HISTORY
WHERE STARTSWITH(TASK_NAME, 'DEV_TELEMETRY.TELEMETRY_TEST.')
ORDER BY DETECTED_TIMESTAMP, TASK_NAME;

SELECT * FROM DEV_TELEMETRY.TELEMETRY.VW_TASK_CURRENT_HEALTH
WHERE STARTSWITH(TASK_NAME, 'DEV_TELEMETRY.TELEMETRY_TEST.');

-- 13.7 Cleanup (DEV only - production telemetry history is never deleted)
--      Disable monitoring first, otherwise suspending the scheduled test tasks opens SUSPENDED alerts.
UPDATE DEV_TELEMETRY.TELEMETRY.TASK_MONITOR_CONFIG
   SET MONITOR_ENABLED = FALSE, ALERT_ENABLED = FALSE, UPDATED_BY = CURRENT_USER(), UPDATED_TIMESTAMP = CURRENT_TIMESTAMP()
 WHERE STARTSWITH(TASK_NAME, 'DEV_TELEMETRY.TELEMETRY_TEST.');
CALL DEV_TELEMETRY.TELEMETRY.SP_EVALUATE_TASK_HEALTH('DEV_TELEMETRY', 'TELEMETRY_TEST');   -- resolves leftovers
ALTER TASK DEV_TELEMETRY.TELEMETRY_TEST.TSK_TEST_SUSPENDED SUSPEND;
ALTER TASK DEV_TELEMETRY.TELEMETRY_TEST.TSK_TEST_OVERDUE   SUSPEND;
-- DROP SCHEMA DEV_TELEMETRY.TELEMETRY_TEST;
-- DELETE FROM DEV_TELEMETRY.TELEMETRY.TASK_MONITOR_CONFIG WHERE STARTSWITH(TASK_NAME, 'DEV_TELEMETRY.TELEMETRY_TEST.');


/* =====================================================================================
   14. OPERATIONS QUERIES
   ===================================================================================== */

-- 14.1 RUNNING right now (live - not stored as a fact)
SELECT DATABASE_NAME || '.' || SCHEMA_NAME || '.' || NAME AS TASK_NAME,
       QUERY_START_TIME,
       DATEDIFF('second', QUERY_START_TIME, CURRENT_TIMESTAMP()) AS RUNNING_SECONDS
FROM TABLE(<LANDING_DB>.INFORMATION_SCHEMA.TASK_HISTORY(
       SCHEDULED_TIME_RANGE_START => DATEADD('hour', -6, CURRENT_TIMESTAMP())))
WHERE STATE = 'EXECUTING';

-- 14.2 Coverage: load tasks that exist but are NOT monitored (target: 36 of 36 covered)
SHOW TASKS IN SCHEMA <LANDING_DB>.<LANDING_SCHEMA>;
SELECT UPPER("database_name" || '.' || "schema_name" || '.' || "name") AS UNMONITORED_TASK
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID())) t
WHERE NOT EXISTS (SELECT 1 FROM DEV_TELEMETRY.TELEMETRY.TASK_MONITOR_CONFIG c
                  WHERE c.TASK_NAME = UPPER(t."database_name" || '.' || t."schema_name" || '.' || t."name"));

-- 14.3 Loaders not publishing row counts (return-value standard not applied)
SELECT TASK_NAME, COUNT(*) AS SUCCEEDED_RUNS, COUNT_IF(ROWS_INSERTED IS NULL) AS RUNS_WITHOUT_COUNTS
FROM DEV_TELEMETRY.TELEMETRY.TASK_EXECUTION_HISTORY
WHERE STATE = 'SUCCEEDED' AND SCHEDULED_TIME >= DATEADD('day', -1, CURRENT_TIMESTAMP())
GROUP BY TASK_NAME
HAVING COUNT_IF(ROWS_INSERTED IS NULL) > 0;

-- 14.4 Daily health snapshot for standup
SELECT HEALTH_STATUS, COUNT(*) AS TASKS
FROM DEV_TELEMETRY.TELEMETRY.VW_TASK_CURRENT_HEALTH
GROUP BY HEALTH_STATUS
ORDER BY TASKS DESC;

-- 14.5 Baseline readiness: which statistical monitors are still in cold start
SELECT c.TASK_NAME, c.METRIC_NAME, c.MIN_SAMPLE_COUNT,
       COUNT(DISTINCT m.METRIC_HISTORY_ID) AS SAMPLES_IN_WINDOW,
       MAX(b.CALCULATED_TIMESTAMP) AS LAST_BASELINE
FROM DEV_TELEMETRY.TELEMETRY.TASK_MONITOR_CONFIG c
LEFT JOIN DEV_TELEMETRY.TELEMETRY.TASK_METRIC_HISTORY m
       ON m.TASK_NAME = c.TASK_NAME AND m.METRIC_NAME = c.METRIC_NAME
      AND m.OBSERVED_TIMESTAMP >= DATEADD('day', -c.BASELINE_WINDOW_DAYS, CURRENT_TIMESTAMP())
LEFT JOIN DEV_TELEMETRY.TELEMETRY.TASK_METRIC_BASELINE b
       ON b.TASK_NAME = c.TASK_NAME AND b.METRIC_NAME = c.METRIC_NAME
WHERE c.LIMIT_TYPE = 'ZSCORE'
GROUP BY c.TASK_NAME, c.METRIC_NAME, c.MIN_SAMPLE_COUNT
ORDER BY SAMPLES_IN_WINDOW;
