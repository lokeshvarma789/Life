# BUILD SPEC FOR COCO — Virtuous Telemetry, Phase 1
## Snowflake Tasks + Snowflake History → Telemetry Collector → TASK_EXECUTION_HISTORY → TASK_METRIC_HISTORY

---

## 0. YOUR ROLE AND HOW TO WORK

You are a senior Snowflake data engineer building a production-grade telemetry framework for a task-based replication pipeline. I am the engineer who owns the work and will run and review everything you write. Build exactly what this spec describes. Where something is a design decision, the reason is written next to it, so keep the behaviour even if you would phrase the SQL differently.

**Build in 6 stages, one workspace file per stage:**

| Stage | File | Runs as |
|---|---|---|
| 1 | `01_foundation.sql` | admin roles (SYSADMIN / SECURITYADMIN / ACCOUNTADMIN) |
| 2 | `02_tables.sql` | `DEV_TELEMETRY_ENGINEER` |
| 3 | `03_collector.sql` | `DEV_TELEMETRY_ENGINEER` |
| 4 | `04_test_tasks.sql` | `DEV_TELEMETRY_ENGINEER` (TEST ONLY) |
| 5 | `05_test_collector.sql` | `DEV_TELEMETRY_ENGINEER` (test suite) |
| 6 | `06_operations.sql` | engineer / reader (runbook queries) |

**After each stage, stop and reply with:**
1. The file content.
2. A list of the objects it creates.
3. The verification queries I should run, and the result I should expect.
4. Any place where Snowflake behaviour could differ from this spec, and how we would know.

Then wait for me to say "continue". Do not start the next stage early.

**Non-negotiable engineering rules:**
1. Fully qualify every object name (`DEV_TELEMETRY.TELEMETRY.X`). Never rely on the session's current database or schema inside procedures, views or tasks.
2. **Tables and streams:** use `CREATE ... IF NOT EXISTS`, never `CREATE OR REPLACE`. Replacing would destroy history, and for a stream it would reset the offset. Seed rows go in with `MERGE ... WHEN NOT MATCHED`, so re-running a file never duplicates or overwrites tuned values.
3. Procedures, functions, views and tasks can use `CREATE OR REPLACE`.
4. **Procedures:** `LANGUAGE SQL` (Snowflake Scripting) and `EXECUTE AS CALLER`. The single exception is the return-value parser, which is a JavaScript UDF. Why caller's rights: `SHOW` + `RESULT_SCAN`, session variables and the task's functional role all behave predictably.
5. No tunable number is hard-coded. Every one is read from `TELEMETRY_SETTINGS`, falls back to a default, and is clamped to a safe range.
6. Every task is created **suspended**. Resuming one is a separate, deliberate step.
7. Never put credentials, passwords or personal logins anywhere. Everything is owned and run by a functional role.
8. **Snowflake Scripting hygiene:**
   - Inside SQL, reference variables as `:var`. Inside scripting expressions (IF, assignments), use the bare name.
   - Never bind VARIANT/OBJECT/ARRAY variables into SQL. Build JSON with `SELECT OBJECT_CONSTRUCT(...) INTO :v` or `::VARCHAR` strings, and pass strings.
   - After a function call, use `GET(expr, 'key')` instead of `expr:key`, so a path is never mistaken for a bind variable.
   - DDL (including `CREATE TEMPORARY TABLE`) commits implicitly, so run all of it **before** `BEGIN TRANSACTION`.
   - Use `BEGIN TRANSACTION` (not bare `BEGIN`) for transactions.
   - User exception codes must be between -20000 and -20999.
   - Wrap calls that fail outside a task (`SYSTEM$TASK_RUNTIME_INFO`, `SYSTEM$SET_RETURN_VALUE`) in their own `BEGIN ... EXCEPTION WHEN OTHER THEN ... END` blocks.
   - Use no apostrophes in comments inside `$$` bodies, and no semicolons inside string literals in DECLARE sections.
9. Comments explain **why**, not what.
10. Don't add features that aren't in this spec. If you think something is missing, list it as a suggestion at the end of the stage.
11. If Snowflake rejects something or behaves differently from section 2, tell me exactly what happened and propose the smallest fix. Don't silently redesign.

---

## 1. BUSINESS CONTEXT (why this exists)

**The pipeline being monitored (built by another engineer; not ready yet):**

```
Virtuous CRM ──► Snowflake data share ──► 36 streams (one per source table)
   ──► root / monitor task ──► 36 child load tasks (MERGE into landing CT tables)
   ──► C360 ──► IM view ──► Talend ──► Virtuous API
```

**The telemetry architecture (from the architect's slide):**

```
Snowflake Tasks + Snowflake History
      │
      ▼
TELEMETRY COLLECTOR ──► TASK_EXECUTION_HISTORY ──► TASK_METRIC_HISTORY      ◄── PHASE 1 (this build)
      │
      ▼  (Phase 2 - NOT in this build)
TASK_MONITOR_CONFIG + Baseline Calculator ──► TASK_METRIC_BASELINE
      ──► Health Monitor ──► ALERT_HISTORY ──► Notifications + Operational Views
```

**Phase 1 scope:** everything up to and including `TASK_METRIC_HISTORY`, plus the control tables, views and tasks needed to run it safely in production.

**Out of scope:** monitor config, baselines, alert rules, alert history, notifications, and data-quality checks on table contents.

**Test data:** the real Virtuous pipeline doesn't exist yet, so Stage 4 builds a sample task graph in schema `TEL_TEST`. It produces every kind of task history the collector must handle. At go-live, the only change is to add a scope row and repoint one view.

---

## 2. SNOWFLAKE FACTS THIS DESIGN RELIES ON (checked against Snowflake docs, Nov 2026)

**`INFORMATION_SCHEMA.TASK_HISTORY()` table function**
- Returns history for the **whole account** (filtered by privileges): the last 7 days, plus scheduled runs up to 8 days ahead.
- Arguments: `SCHEDULED_TIME_RANGE_START/END` (TIMESTAMP_LTZ; must fall within the last 7 days, otherwise an error), `RESULT_LIMIT` (1–10000, **default 100**), `TASK_NAME` (unqualified), **`DATABASE_NAME`, `SCHEMA_NAME`** (applied **before** RESULT_LIMIT), `ERROR_ONLY`, `ROOT_TASK_ID`, `GRAPH_RUN_GROUP_ID`.
- If more rows match than the limit allows, only the most recent are returned. Results come in descending `SCHEDULED_TIME`.
- With END = now, it returns completed and currently running tasks.
- Columns used:
  - Identity: `QUERY_ID`, `NAME`, `DATABASE_NAME`, `SCHEMA_NAME`.
  - Outcome: `STATE`, `ERROR_CODE` (NUMBER), `ERROR_MESSAGE`.
  - Timing: `SCHEDULED_TIME`, `QUERY_START_TIME`, `NEXT_SCHEDULED_TIME`, `COMPLETED_TIME`.
  - Graph: `ROOT_TASK_ID`, `GRAPH_VERSION`, `RUN_ID`, `GRAPH_RUN_GROUP_ID`.
  - Run details: `RETURN_VALUE`, `SCHEDULED_FROM`, `ATTEMPT_NUMBER`.
- STATE values: `SCHEDULED`, `EXECUTING`, `SUCCEEDED`, `FAILED`, `FAILED_AND_AUTO_SUSPENDED`, `CANCELLED`, `SKIPPED`.
- `SCHEDULED_FROM` values: `SCHEDULE`, `EXECUTE_TASK`, `MANUAL RETRY`, `AUTOMATIC RETRY`, `TRIGGER`.
- `GRAPH_RUN_GROUP_ID` + `ATTEMPT_NUMBER` identifies one graph run. `RUN_ID` is not unique across retries.
- Privileges: OWNERSHIP, MONITOR or OPERATE on the task, or the global MONITOR EXECUTION, **plus USAGE on the database and schema**. Without USAGE, `DATABASE_NAME`/`SCHEMA_NAME` come back NULL.
- After a cloud-services failure, one execution can appear twice, with different ids and scheduled times.

**`SNOWFLAKE.ACCOUNT_USAGE.TASK_HISTORY` view**
- 365 days of history, latency up to 45 minutes.
- Terminal states only: SUCCEEDED, FAILED, CANCELLED, SKIPPED.
- **No `NEXT_SCHEDULED_TIME`.** `ERROR_CODE` is VARCHAR.
- Requires `SNOWFLAKE.USAGE_VIEWER` (or imported privileges).

**`SHOW TASKS [IN SCHEMA db.schema]`**
- Output columns are lowercase and must be double-quoted: `"id"`, `"name"`, `"database_name"`, `"schema_name"`, `"owner"`, `"warehouse"`, `"schedule"`, `"predecessors"`, `"state"`, `"definition"`, `"condition"`, `"allow_overlapping_execution"`, `"config"`, `"task_relations"`, `"last_suspended_reason"`, `"created_on"`, `"last_committed_on"`, `"last_suspended_on"`.
  - `"predecessors"`: JSON array of fully qualified names; empty for a root.
  - `"state"`: `started` / `suspended`.
  - `"task_relations"`: JSON with fields such as `Predecessors`, `FinalizerTask`, `FinalizedRootTask`.
- `CREATE OR REPLACE TASK` creates a **new id**.
- Post-process with `RESULT_SCAN(LAST_QUERY_ID())`.

**`INFORMATION_SCHEMA.COMPLETE_TASK_GRAPHS()`**
- Completed graph runs from the **last 60 minutes**.
- Arguments: `RESULT_LIMIT`, `ROOT_TASK_NAME`, `ERROR_ONLY`, `GRAPH_RUN_GROUP_ID`.
- Columns include `STATE`, `FIRST_ERROR_TASK_NAME`, `ATTEMPT_NUMBER`, `GRAPH_RUN_GROUP_ID`.
- No row is returned if the root was SKIPPED.

**`EXECUTE TASK`**
- A suspended root or standalone task can be executed; suspended children are skipped, so children must be resumed.
- Asynchronous unless `AWAIT` is used.
- `EXECUTE TASK <root> RETRY LAST` needs three things: the last graph run FAILED or was CANCELLED, the graph hasn't been modified since, and the run is within 14 days. It creates `ATTEMPT_NUMBER + 1` with the same `GRAPH_RUN_GROUP_ID`, and `RETRY LAST` doesn't support `AWAIT`.

**Task functions and limits**
- `SYSTEM$SET_RETURN_VALUE(string)`: maximum 10 KB. Works in a task body (`CALL SYSTEM$SET_RETURN_VALUE('...')`) or from a procedure called by the task. It appears as `TASK_HISTORY.RETURN_VALUE`.
- `SYSTEM$TASK_RUNTIME_INFO('CURRENT_TASK_NAME' | 'CURRENT_TASK_GRAPH_RUN_GROUP_ID')`: only valid while running inside a task.
- Finalizer tasks (`FINALIZE = <root>`) run after every graph run, success or failure.
- Children can have a `WHEN SYSTEM$STREAM_HAS_DATA('stream')` clause. When it is false, the run is SKIPPED and no warehouse is used.

---

## 3. ENVIRONMENT AND NAMING

| Item | Name |
|---|---|
| Database | `DEV_TELEMETRY` (promote by replacing the `DEV_` prefix) |
| Framework schema | `DEV_TELEMETRY.TELEMETRY` (WITH MANAGED ACCESS) |
| Test schema | `DEV_TELEMETRY.TEL_TEST` (TEST ONLY) |
| Warehouse | `DEV_TELEMETRY_WH`: XSMALL, AUTO_SUSPEND 60, AUTO_RESUME, INITIALLY_SUSPENDED |
| Resource monitor | `DEV_TELEMETRY_RM`: 40 credits/month, notify at 75% and 90%, suspend at 100% |
| Owner role | `DEV_TELEMETRY_ENGINEER` |
| Read role | `DEV_TELEMETRY_READER` (views only) |

**Conventions:**
- `TASK_NAME` is always the **UPPER fully qualified** `DB.SCHEMA.TASK`.
- Timestamps are `TIMESTAMP_LTZ`, the same type `TASK_HISTORY` uses.
- NULL row counts mean "the loader did not report". Never write a fake 0.
- PK/FK constraints are informational (Snowflake doesn't enforce them). Idempotency comes from design, not constraints.

---

## STAGE 1 — `01_foundation.sql` (admin-run)

**1. As SYSADMIN:** create the warehouse, the database, schema `TELEMETRY` WITH MANAGED ACCESS, and schema `TEL_TEST`. Give each a comment explaining its purpose.

**2. As ACCOUNTADMIN:** create the resource monitor and attach it to the warehouse.
*Why: a hard cost ceiling for telemetry and test traffic.*

**3. As SECURITYADMIN:**
- Create both roles.
- Grant the reader role to the engineer role, and the engineer role to SYSADMIN (keeps the hierarchy intact).
- Grant the engineer role to `<YOUR_SNOWFLAKE_USER>`. This is the only placeholder.
- On the warehouse, grant USAGE and OPERATE to the engineer role. On the database, grant USAGE to the engineer role.
- Grant **OWNERSHIP** on both schemas to the engineer role, with COPY CURRENT GRANTS.
- Reader: USAGE on the database and the `TELEMETRY` schema, and `SELECT ON FUTURE VIEWS IN SCHEMA TELEMETRY`.
  *Why: readers see views, never base tables.*

**4. As ACCOUNTADMIN:**
- `GRANT EXECUTE TASK ON ACCOUNT`. *Why: needed to run any task.*
- `GRANT MONITOR EXECUTION ON ACCOUNT`. *Why: lets TASK_HISTORY return runs of tasks this role doesn't own, i.e. the Virtuous tasks at go-live.*
- Optional: `GRANT DATABASE ROLE SNOWFLAKE.USAGE_VIEWER`, only for the ACCOUNT_USAGE backfill mode. Mark it optional in a comment.

**5. Hand-off check:** switch to the engineer role and warehouse, then `SELECT CURRENT_ROLE(), CURRENT_WAREHOUSE()` and `SHOW GRANTS TO ROLE DEV_TELEMETRY_ENGINEER`.

---

## STAGE 2 — `02_tables.sql` (8 tables, all `CREATE TABLE IF NOT EXISTS`, every column with a COMMENT)

### 2.1 `TASK_EXECUTION_HISTORY` — one row per terminal task run (the architect's table)
*Why: an immutable fact per run, kept 13 months for RCA and audit. It is append-only and never updated; only the retention purge deletes.*

**Architect's columns:**

| Column | Type | Null | Meaning |
|---|---|---|---|
| TASK_RUN_ID | VARCHAR(32) | NOT NULL | Deterministic key (see FN_TASK_RUN_ID). PK. |
| TASK_NAME | VARCHAR(500) | NOT NULL | UPPER `DB.SCHEMA.TASK` |
| GRAPH_NAME | VARCHAR(500) | | Root task name (from TASK_CATALOG via ROOT_TASK_ID); equals TASK_NAME for a standalone task |
| STATE | VARCHAR(30) | NOT NULL | Terminal states only |
| ERROR_CODE | VARCHAR(50) | | |
| ERROR_MESSAGE | VARCHAR(4000) | | First 4000 chars |
| SCHEDULED_TIME | TIMESTAMP_LTZ | NOT NULL | |
| START_TIME | TIMESTAMP_LTZ | | = QUERY_START_TIME; NULL when SKIPPED |
| END_TIME | TIMESTAMP_LTZ | | = COMPLETED_TIME |
| DURATION_SECONDS | NUMBER(18,3) | | END − START in seconds, millisecond precision |
| ROWS_INSERTED | NUMBER(38,0) | | From the loader return value; NULL = not reported |
| ROWS_UPDATED | NUMBER(38,0) | | Same |
| ROWS_DELETED | NUMBER(38,0) | | Same |
| SOURCE_NAME | VARCHAR(500) | | Lineage: loader value first, registry second |
| STREAM_NAME | VARCHAR(500) | | Same |
| TARGET_NAME | VARCHAR(500) | | Same |
| COLLECTED_TIMESTAMP | TIMESTAMP_LTZ | NOT NULL | When the collector inserted it |

**Proposed additions** (free Snowflake metadata; mark them "PROPOSED" in comments):

| Column | Type | Null | Meaning |
|---|---|---|---|
| PIPELINE_NAME | VARCHAR(100) | | From the collect scope |
| QUERY_ID | VARCHAR(100) | | Jumps to the query profile; NULL when SKIPPED |
| ROOT_TASK_ID | VARCHAR(100) | | |
| GRAPH_VERSION | NUMBER(38,0) | | Changes when the graph definition changes (useful in RCA) |
| GRAPH_RUN_GROUP_ID | VARCHAR(100) | | Shared by all tasks in one graph run and its retries |
| GRAPH_RUN_ID | NUMBER(38,0) | | = TASK_HISTORY.RUN_ID |
| ATTEMPT_NUMBER | NUMBER(10,0) | | 1 = first run, 2+ = retry |
| SCHEDULED_FROM | VARCHAR(30) | | |
| NEXT_SCHEDULED_TIME | TIMESTAMP_LTZ | | For schedule-aware OVERDUE in Phase 2; NULL from ACCOUNT_USAGE |
| START_DELAY_SECONDS | NUMBER(18,3) | | START − SCHEDULED (warehouse queuing signal) |
| RETURN_VALUE | VARCHAR(10000) | | Raw payload |
| RETURN_VALUE_STATUS | VARCHAR(20) | | `ABSENT` / `PARSED` / `NO_COUNTS` / `INVALID_JSON` / `INVALID_VALUES` |
| RETURN_VALUE_ISSUES | VARCHAR(2000) | | Human-readable contract problems |
| CUSTOM_METRICS | OBJECT | | Validated loader metrics `{NAME: number}` |
| LOADER_EXECUTION_ID | VARCHAR(100) | | Loader's own run id; joins to the loader audit log |
| COLLECTOR_RUN_ID | VARCHAR(36) | NOT NULL | Which collector run wrote the row |
| HISTORY_SOURCE | VARCHAR(30) | NOT NULL | `INFORMATION_SCHEMA` or `ACCOUNT_USAGE` |

### 2.2 `TASK_METRIC_HISTORY` — one row per run per metric (tall/narrow, the architect's table)
*Why: a new measurement means new rows, never a new column. Baselines in Phase 2 work on (task, metric) pairs.*

| Column | Type | Null | Meaning |
|---|---|---|---|
| METRIC_HISTORY_ID | NUMBER(38,0) IDENTITY(1,1) | | PROPOSED surrogate PK |
| TASK_RUN_ID | VARCHAR(32) | | FK → execution history. NULL only for future non-run metrics. |
| TASK_NAME | VARCHAR(500) | NOT NULL | |
| METRIC_NAME | VARCHAR(100) | NOT NULL | |
| METRIC_VALUE | NUMBER(38,6) | | |
| UNIT_OF_MEASURE | VARCHAR(30) | | SECONDS / ROWS / FLAG / COUNT / BYTES |
| DIMENSION_KEY | VARCHAR(200) | NOT NULL | DEFAULT `'RUN'` (Phase 2 adds `DAY`, `TABLE:<x>`) |
| OBSERVED_TIMESTAMP | TIMESTAMP_LTZ | NOT NULL | Run END_TIME (SCHEDULED_TIME if no END_TIME) |
| COLLECTED_TIMESTAMP | TIMESTAMP_LTZ | NOT NULL | |
| METRIC_SOURCE | VARCHAR(30) | NOT NULL | PROPOSED: `TASK_HISTORY` / `RETURN_VALUE` / `DERIVED` / `RETURN_VALUE_CUSTOM` |
| RUN_STATE | VARCHAR(30) | | PROPOSED: lets baselines use SUCCEEDED runs only, without a join |
| PIPELINE_NAME | VARCHAR(100) | | PROPOSED |
| COLLECTOR_RUN_ID | VARCHAR(36) | NOT NULL | PROPOSED |

### 2.3 `TELEMETRY_SETTINGS` — every tunable in one place
*Why: no magic numbers in code; tuning is an UPDATE, not a redeploy.*

Columns: `SETTING_NAME` VARCHAR(100) PK, `SETTING_VALUE` VARIANT NOT NULL, `DESCRIPTION` VARCHAR(1000), `UPDATED_BY` (DEFAULT CURRENT_USER()), `UPDATED_TIMESTAMP` (DEFAULT CURRENT_TIMESTAMP()).

Seed with a MERGE that inserts missing keys only:

| Setting | Value | Why |
|---|---|---|
| ENVIRONMENT | "DEV" | Label shown in the health view |
| COLLECT_OVERLAP_MINUTES | 360 | Re-read 6 h before each watermark. Must exceed the longest task run, so long runs are never missed |
| COLLECT_INITIAL_LOOKBACK_HOURS | 167 | A new scope reads 7 days minus 1 h (the INFORMATION_SCHEMA limit) |
| COLLECT_SLICE_MINUTES | 360 | Size of each TASK_HISTORY call; halved automatically on limit hits |
| COLLECT_RESULT_LIMIT | 10000 | The default would be only 100 |
| COLLECT_STALE_RUN_MINUTES | 30 | A RUNNING log row older than this is marked ABANDONED |
| ACCOUNT_USAGE_LATENCY_MINUTES | 60 | Backfill never reads the newest hour |
| COLLECTOR_STALE_ALERT_MINUTES | 45 | Health view reports DOWN after this |
| HISTORY_RETENTION_DAYS | 400 | 13 months, for year-over-year comparison |
| METRIC_RETENTION_DAYS | 400 | |
| COLLECTOR_LOG_RETENTION_DAYS | 90 | |
| CATALOG_DROPPED_RETENTION_DAYS | 400 | Old runs must still resolve to a task name |

### 2.4 `TELEMETRY_COLLECT_SCOPE` — what to collect, and where each scope is up to
*Why: onboarding a pipeline is one INSERT, and a per-scope watermark makes outages self-heal.*

**Columns:**
- `SCOPE_ID` NUMBER IDENTITY, PK
- `PIPELINE_NAME` VARCHAR(100) NOT NULL
- `DATABASE_NAME`, `SCHEMA_NAME` VARCHAR(255) NOT NULL. UPPER, exactly as Snowflake stores them.
- `TASK_NAME_LIKE` VARCHAR(255) NOT NULL DEFAULT '%'. An ILIKE pattern, so one schema can hold both monitored and unmonitored tasks.
- `ENABLED` BOOLEAN NOT NULL DEFAULT TRUE
- **Collector-maintained:** `WATERMARK_TS`, `LAST_SUCCESS_AT`, `LAST_COLLECTOR_RUN_ID`, `LAST_ERROR` VARCHAR(4000), `LAST_ERROR_AT`, `CATALOG_REFRESHED_AT`, `CATALOG_ERROR` VARCHAR(4000), `CATALOG_ERROR_AT`
- **Audit:** `NOTES` VARCHAR(1000) (don't name it COMMENT), `CREATED_BY`, `CREATED_TIMESTAMP`, `UPDATED_BY`, `UPDATED_TIMESTAMP`
- Informational UNIQUE on (DATABASE_NAME, SCHEMA_NAME, TASK_NAME_LIKE)

**Seed with MERGE:** `('TELEMETRY', 'DEV_TELEMETRY', 'TELEMETRY', 'TSK_TEL%')`.
*Why: the framework monitors its own tasks like any other.*

### 2.5 `METRIC_DEFINITION` — governed metric dictionary
**Columns:** `METRIC_NAME` PK, `UNIT_OF_MEASURE` NOT NULL, `METRIC_SOURCE` NOT NULL, `IS_CORE` BOOLEAN NOT NULL, `IS_ACTIVE` BOOLEAN NOT NULL DEFAULT TRUE, `DESCRIPTION` NOT NULL, `UPDATED_BY`, `UPDATED_TIMESTAMP`.

**Seed with MERGE.** When a row is matched, update unit, source, is_core and description, because code owns the definition of core metrics. Never overwrite `IS_ACTIVE`; operations owns that.

| METRIC_NAME | Unit | Source | Core | Rule |
|---|---|---|---|---|
| DURATION_SECONDS | SECONDS | TASK_HISTORY | Y | Executed runs only (not SKIPPED) |
| START_DELAY_SECONDS | SECONDS | TASK_HISTORY | Y | Executed runs only |
| ROWS_INSERTED | ROWS | RETURN_VALUE | Y | SUCCEEDED runs only |
| ROWS_UPDATED | ROWS | RETURN_VALUE | Y | SUCCEEDED runs only |
| ROWS_DELETED | ROWS | RETURN_VALUE | Y | SUCCEEDED runs only |
| ROWS_TOTAL | ROWS | DERIVED | Y | SUCCEEDED and status PARSED only: ins + upd + del |
| RUN_FAILED | FLAG | TASK_HISTORY | Y | 1 = FAILED / FAILED_AND_AUTO_SUSPENDED / CANCELLED, 0 = SUCCEEDED, not emitted for SKIPPED. AVG = failure rate |
| ROWS_REJECTED | ROWS | RETURN_VALUE_CUSTOM | N | Optional loader metric |
| BYTES_LOADED | BYTES | RETURN_VALUE_CUSTOM | N | Optional loader metric |

### 2.6 `TASK_CATALOG` — persisted SHOW TASKS snapshot, one row per task id ever seen
*Why: TASK_HISTORY only gives ROOT_TASK_ID. The catalog turns it into GRAPH_NAME, and it keeps rows for dropped or recreated tasks so old runs can still be named.*

**Columns:**
- Identity: `TASK_ID` VARCHAR(100) PK, `TASK_NAME` VARCHAR(500), `DATABASE_NAME`, `SCHEMA_NAME`, `TASK_SHORT_NAME`, `PIPELINE_NAME`
- Definition: `OWNER_ROLE`, `WAREHOUSE_NAME`, `SCHEDULE`, `PREDECESSORS` ARRAY, `IS_ROOT` BOOLEAN (no predecessors and not a finalizer), `FINALIZES_ROOT_TASK` VARCHAR(500), `CONDITION_TEXT` VARCHAR(2000), `DEFINITION` VARCHAR, `DEFINITION_HASH` VARCHAR(32) (MD5 of DEFINITION; a change means someone changed the task), `ALLOW_OVERLAPPING_EXECUTION` VARCHAR(10), `TASK_CONFIG` VARCHAR(4000), `TASK_RELATIONS` VARIANT
- State: `TASK_STATE` VARCHAR(20), `LAST_SUSPENDED_REASON` VARCHAR(100), `TASK_CREATED_ON`, `LAST_COMMITTED_ON`, `LAST_SUSPENDED_ON`
- Tracking: `FIRST_SEEN_AT` NOT NULL, `LAST_SEEN_AT` NOT NULL, `LAST_SEEN_BY_RUN_ID` VARCHAR(36), `IS_DROPPED` BOOLEAN NOT NULL DEFAULT FALSE, `DROPPED_DETECTED_AT`

### 2.7 `TELEMETRY_COLLECTOR_RUN` — audit row per collector execution
*Why: the run log is the source for collector health and the Phase 2 watchdog, and it proves what each run did.*

**Columns:**
- `COLLECTOR_RUN_ID` VARCHAR(36) PK
- `RUN_MODE`: `INCREMENTAL` / `WINDOW` / `ACCOUNT_USAGE` / `REBUILD_METRICS`
- `STATUS`: `RUNNING` / `SUCCEEDED` / `SUCCEEDED_WITH_WARNINGS` / `PARTIAL` / `FAILED` / `ABANDONED`
- `STARTED_AT` NOT NULL, `ENDED_AT`, `WINDOW_START`, `WINDOW_END`
- Counters, each NUMBER DEFAULT 0:
  - `SCOPES_PROCESSED`, `SCOPES_FAILED`
  - `SLICES_QUERIED`, `RESULT_LIMIT_HITS`, `HISTORY_ROWS_READ`
  - `TERMINAL_RUNS_SEEN`, `NONTERMINAL_RUNS_SEEN`, `RUNS_ALREADY_COLLECTED`
  - `RUNS_INSERTED`, `METRICS_INSERTED`, `CATALOG_TASKS_SEEN`, `WARNING_COUNT`
- `WARNINGS` VARCHAR(16000), `ERROR_MESSAGE` VARCHAR(4000)
- `INVOKED_BY_TASK` VARCHAR(500) (NULL when a person calls it), `SESSION_USER`, `SESSION_ROLE`, `WAREHOUSE_NAME`

### 2.8 `TELEMETRY_COLLECTOR_LOCK` — one-row mutex
Columns: `LOCK_NAME` PK, `LOCKED_BY_RUN_ID`, `LOCKED_AT`. Seed the row `'TASK_TELEMETRY_COLLECTOR'` with MERGE.

*Why: the collector's first statement in its write transaction is an UPDATE of this row. Snowflake holds that lock until COMMIT, so a concurrent collector waits, then its anti-join sees the committed rows. Without this, two collectors could insert the same run twice, because PKs aren't enforced.*

### 2.9 Verify section
1. List the TELEMETRY tables with row counts.
2. Show the settings and the metric definitions.
3. **Schema guard query:** using INFORMATION_SCHEMA.COLUMNS, return `OK` only if TASK_EXECUTION_HISTORY has COLLECTOR_RUN_ID, HISTORY_SOURCE, RETURN_VALUE_STATUS and CUSTOM_METRICS, and TASK_METRIC_HISTORY has METRIC_SOURCE, RUN_STATE and COLLECTOR_RUN_ID. Otherwise return `STOP - older table exists`.
   *Why: `IF NOT EXISTS` silently keeps an older table from a previous draft.*

---

## STAGE 3 — `03_collector.sql`

### 3.1 `FN_TASK_RUN_ID(TASK_FQN VARCHAR, GRAPH_RUN_GROUP_ID VARCHAR, ATTEMPT_NUMBER NUMBER, RUN_ID NUMBER, SCHEDULED_TIME TIMESTAMP_LTZ) RETURNS VARCHAR IMMUTABLE` (SQL UDF)

```
MD5( UPPER(TASK_FQN) || '|' ||
     COALESCE(GRAPH_RUN_GROUP_ID, COALESCE(TO_VARCHAR(RUN_ID),'') || '@' ||
              COALESCE(TO_VARCHAR(DATE_PART(EPOCH_NANOSECOND, SCHEDULED_TIME)),'')) || '|' ||
     COALESCE(TO_VARCHAR(ATTEMPT_NUMBER), '1') )
```

*Why: Snowflake documents GRAPH_RUN_GROUP_ID + ATTEMPT_NUMBER as a run's identity, and adding the task name makes it unique per task. Collecting the same run twice produces the same key, and a retry produces a new key. The fallback uses epoch nanoseconds, so it doesn't depend on the timezone.*

### 3.2 `FN_PARSE_RETURN_VALUE(RAW VARCHAR) RETURNS OBJECT LANGUAGE JAVASCRIPT IMMUTABLE` — return-value contract v1

*Why JavaScript: it can be unit-tested without running a task, it handles every malformed shape, and it must never throw.*

**The contract a loader follows**, via `SYSTEM$SET_RETURN_VALUE` (maximum 10 KB):

```json
{"contract_version":1,"inserted":100,"updated":15,"deleted":5,
 "source":"DB.SCH.OBJ","stream":"DB.SCH.STR","target":"DB.SCH.TBL",
 "execution_id":"loader-run-id","metrics":{"rows_rejected":0,"bytes_loaded":123}}
```

**Parsing rules:**
- Keys are **case-insensitive**; the first occurrence wins. `rows_inserted` / `rows_updated` / `rows_deleted` are aliases.
- A key whose value is JSON `null` counts as absent.
- **A count is valid only if it is a non-negative integer.** It may be a JSON number or a numeric string (`"12"`). Decimals, negatives, booleans and NaN are invalid; the invalid count is dropped and an issue recorded. Upper limit: 9007199254740991.
- **Status:**
  - `ABSENT`: null, empty or whitespace input
  - `INVALID_JSON`: unparsable, or JSON that is not an object (an array or a scalar)
  - `NO_COUNTS`: an object with no count key
  - `INVALID_VALUES`: any present count is invalid
  - `PARSED`: otherwise; `rows_total` = sum of the valid counts, and only set when PARSED
- **Lineage:** `source`, `stream`, `target` and `execution_id` are kept if they are a non-empty string or number, trimmed, maximum 500 characters. `contract_version` is kept as a string.
- **Custom metrics** (`metrics` must be an object, otherwise an issue is recorded):
  - Names are upper-cased and must match `^[A-Z][A-Z0-9_]{0,63}$`.
  - Values must be finite numbers or numeric strings, with |v| < 1e30.
  - Duplicates are ignored, and at most 50 are kept; extras are dropped with an issue.
  - Custom-metric problems add issues but **do not change the status**.
- **Output:** an OBJECT with keys `status`, `inserted`, `updated`, `deleted`, `rows_total`, `source`, `stream`, `target`, `execution_id`, `contract_version`, `metrics`, `issues` (array of strings). **A key is present only when it has a value**, so `p:x` is SQL NULL when the value is missing.

**Self-test cases** (they must all pass; Stage 5 test T01.1 uses exactly these):

| Input | status | ins | upd | del | total | #metrics | has issues |
|---|---|---|---|---|---|---|---|
| `{"inserted":100,"updated":15,"deleted":5}` | PARSED | 100 | 15 | 5 | 120 | 0 | no |
| `{"rows_inserted":7,"rows_updated":0,"rows_deleted":1}` | PARSED | 7 | 0 | 1 | 8 | 0 | no |
| `{"INSERTED":3,"Updated":2}` | PARSED | 3 | 2 | – | 5 | 0 | no |
| `{"inserted":"12","deleted":"0"}` | PARSED | 12 | – | 0 | 12 | 0 | no |
| `{"inserted":9}` | PARSED | 9 | – | – | 9 | 0 | no |
| `{"inserted":null,"updated":4}` | PARSED | – | 4 | – | 4 | 0 | no |
| NULL | ABSENT | – | – | – | – | 0 | no |
| `'   '` | ABSENT | – | – | – | – | 0 | no |
| `load finished OK` | INVALID_JSON | – | – | – | – | 0 | yes |
| `[1,2,3]` | INVALID_JSON | – | – | – | – | 0 | yes |
| `42` | INVALID_JSON | – | – | – | – | 0 | yes |
| `{"metrics":{"finalizer_ran":1}}` | NO_COUNTS | – | – | – | – | 1 | no |
| `{"inserted":-5,"updated":1}` | INVALID_VALUES | – | 1 | – | – | 0 | yes |
| `{"inserted":1.5}` | INVALID_VALUES | – | – | – | – | 0 | yes |
| `{"inserted":1,"metrics":{"rows_rejected":2,"bad name":3,"bytes_loaded":"1024","flag":true}}` | PARSED | 1 | – | – | 1 | 2 | yes |
| `{"deleted":2,"metrics":[1,2]}` | PARSED | – | – | 2 | 2 | 0 | yes |

Issue text for the negative case must contain `inserted must be a non-negative integer`.

### 3.3 `V_PIPELINE_OBJECTS` — lineage adapter view
Create it with `CREATE VIEW IF NOT EXISTS`, as an empty stub (`WHERE FALSE`).

Columns: `PIPELINE_NAME`, `LOAD_TASK_NAME` (UPPER FQN), `SOURCE_OBJECT`, `STREAM_OBJECT`, `TARGET_OBJECT`, `ENABLED` BOOLEAN.

*Why IF NOT EXISTS: Stage 4 repoints it to the test registry, and go-live repoints it to the real metadata table. Re-running this file must never undo that repoint. The collector reads pipeline metadata only through this view.*

### 3.4 `SP_REFRESH_TASK_CATALOG(P_COLLECTOR_RUN_ID VARCHAR DEFAULT NULL) RETURNS VARIANT`

Loop over enabled scopes with a cursor FOR loop. For each scope, inside its own BEGIN/EXCEPTION block:

1. Validate DATABASE_NAME and SCHEMA_NAME against `^[A-Z_][A-Z0-9_]*$`. If either fails, raise a declared exception.
   *Why: the names are concatenated into SHOW, so this prevents injection.*
2. `EXECUTE IMMEDIATE 'SHOW TASKS IN SCHEMA ' || db || '.' || sch`.
3. `MERGE INTO TASK_CATALOG` USING `SELECT ... FROM TABLE(RESULT_SCAN(LAST_QUERY_ID())) WHERE "name" ILIKE :v_like`, keyed on TASK_ID:
   - `PREDECESSORS = COALESCE(TRY_PARSE_JSON("predecessors"::VARCHAR)::ARRAY, ARRAY_CONSTRUCT())`
   - `TASK_RELATIONS = TRY_PARSE_JSON("task_relations"::VARCHAR)`
   - `FINALIZES_ROOT_TASK = UPPER(GET(TRY_PARSE_JSON("task_relations"::VARCHAR), 'FinalizedRootTask')::VARCHAR)`
   - `IS_ROOT = ARRAY_SIZE(PREDECESSORS) = 0 AND FINALIZES_ROOT_TASK IS NULL`
   - `DEFINITION_HASH = MD5("definition")`
   - Set `PIPELINE_NAME` from the scope, `LAST_SEEN_AT = v_ts` (one timestamp for the whole call), `LAST_SEEN_BY_RUN_ID`, `IS_DROPPED = FALSE`, `DROPPED_DETECTED_AT = NULL`. On insert, `FIRST_SEEN_AT = v_ts`.
   - Use unqualified column names on the left of SET.
4. Mark dropped tasks: set `IS_DROPPED = TRUE, DROPPED_DETECTED_AT = v_ts` for catalog rows in the same db/schema, matching the pattern, not already dropped, with `LAST_SEEN_AT < v_ts`.
5. On success, set the scope's `CATALOG_REFRESHED_AT = v_ts` and clear `CATALOG_ERROR`. On exception, write `CATALOG_ERROR = LEFT(SQLERRM, 4000)` and `CATALOG_ERROR_AT = v_ts`.

**Never raise out of this procedure.** *Why: the catalog is enrichment, and its failure must never stop fact collection.*

Return an OBJECT with refreshed_at, scopes, scopes_failed, tasks_seen and tasks_marked_dropped.

### 3.5 `VW_TASK_METRIC_DERIVATION` — the ONE place metric rules live
*Why: the collector and the rebuild procedure use identical logic. A new metric is one definition row, one CASE line, and a rebuild.*

**Core metrics:** `TASK_EXECUTION_HISTORY e CROSS JOIN METRIC_DEFINITION d WHERE d.IS_CORE AND d.IS_ACTIVE`. METRIC_VALUE comes from a CASE on `d.METRIC_NAME` that implements the rules in the table in 2.5, cast to NUMBER(38,6).

**Custom metrics:**
- `LATERAL FLATTEN(INPUT => e.CUSTOM_METRICS)` for SUCCEEDED runs: name = `UPPER(f.KEY)`, value = `TRY_TO_DOUBLE(f.VALUE::VARCHAR)`.
- LEFT JOIN METRIC_DEFINITION. Keep a row only where:
  - the name is not a core metric (*a loader can never overwrite a core metric*),
  - `COALESCE(IS_ACTIVE, TRUE)`,
  - the value is not NULL and |value| < 1e30.
- Unit = the definition's unit, otherwise `COUNT`. Source = `RETURN_VALUE_CUSTOM`.

**Output** (UNION ALL, rows with a non-NULL value only): TASK_RUN_ID, TASK_NAME, PIPELINE_NAME, RUN_STATE, METRIC_NAME, METRIC_VALUE, UNIT_OF_MEASURE, DIMENSION_KEY = 'RUN', METRIC_SOURCE, OBSERVED_TIMESTAMP = COALESCE(END_TIME, SCHEDULED_TIME), SCHEDULED_TIME, COLLECTOR_RUN_ID, COLLECTED_TIMESTAMP.

### 3.6 `SP_COLLECT_TASK_TELEMETRY(P_MODE VARCHAR DEFAULT 'INCREMENTAL', P_WINDOW_START TIMESTAMP_LTZ DEFAULT NULL, P_WINDOW_END TIMESTAMP_LTZ DEFAULT NULL) RETURNS VARIANT` — THE COLLECTOR

**Modes:**
- `INCREMENTAL` (what the task runs): per-scope window `[watermark − overlap, now]`.
- `WINDOW`: re-read an explicit window inside the last 167 h. It is idempotent and safe at any time.
- `ACCOUNT_USAGE`: backfill from `SNOWFLAKE.ACCOUNT_USAGE.TASK_HISTORY`, up to 365 days back. The end is clamped to now − ACCOUNT_USAGE_LATENCY_MINUTES.

**Algorithm:**

**Step 0 — identity and validation**
- `v_run_id = UUID_STRING()`, `v_started = CURRENT_TIMESTAMP()`.
- Normalise the mode; an invalid mode raises a declared exception. `HISTORY_SOURCE` = `ACCOUNT_USAGE` or `INFORMATION_SCHEMA`.
- `SELECT SYSTEM$TASK_RUNTIME_INFO('CURRENT_TASK_NAME') INTO :v_invoker_task` in its own exception block; NULL when not inside a task.
- Read all settings in one pivot SELECT: `MAX(IFF(SETTING_NAME = ..., TRY_TO_NUMBER(SETTING_VALUE::VARCHAR), NULL))` with defaults. Then clamp:

  | Setting | Clamp |
  |---|---|
  | overlap | 15–2880 |
  | initial lookback | 1–167 |
  | slice | 5–1440 |
  | result limit | 100–10000 |
  | stale-run minutes | 5–1440 |
  | ACCOUNT_USAGE latency | 45–1440 |

- `v_floor = now − 167 h`.
- For the explicit modes, validate the window: start NOT NULL and start < end; end = LEAST(end or now, now); for WINDOW, start ≥ floor; for ACCOUNT_USAGE, start ≥ now − 365 days and end ≤ now − latency. A violation raises a declared exception.
- Mark old `RUNNING` log rows (older than the stale-run minutes) as `ABANDONED`, with an explanatory ERROR_MESSAGE.
- Insert this run's log row: STATUS `RUNNING`, invoker task, CURRENT_USER/ROLE/WAREHOUSE.

**Step 1 — catalog.** `CALL SP_REFRESH_TASK_CATALOG(:v_run_id)` in an exception block; a failure becomes a warning. Then count `TASK_CATALOG WHERE LAST_SEEN_BY_RUN_ID = :v_run_id`. Any scope whose `CATALOG_ERROR_AT >= v_started` adds a warning.

**Step 2 — staging.** Create two session temp tables, both before any transaction:
- `CREATE OR REPLACE TEMPORARY TABLE TMP_TEL_HISTORY_RAW` with columns SLICE_NO, SCOPE_ID and all the TASK_HISTORY columns listed above. ERROR_CODE is stored as VARCHAR.
- `TMP_TEL_SCOPE_STATUS` with columns SCOPE_ID, WINDOW_START, WINDOW_END, STATUS (OK / SATURATED / FAILED), ROWS_READ, ERROR_TEXT.

**Step 3 — read history, scope by scope** (cursor over enabled scopes). For each scope:

- **INCREMENTAL window:** end = v_started; start = `(COALESCE(WATERMARK_TS, now − initial_hours)) − overlap`; if start < floor, set start = floor. If the watermark itself is older than the floor, add warning `GAP_OVER_7_DAYS`, telling the operator to backfill with ACCOUNT_USAGE mode. For the explicit modes, use the requested window.
- In its own BEGIN/EXCEPTION block:
  1. Validate the identifiers with the same regex; failure raises a declared exception.
  2. **ACCOUNT_USAGE:** one INSERT from the view, filtered on DATABASE_NAME, SCHEMA_NAME and `SCHEDULED_TIME >= start AND < end`. NEXT_SCHEDULED_TIME is NULL.
  3. **Otherwise:** slice loop:

     ```
     slice_start = start; cur = slice_minutes
     WHILE slice_start < end:
        slice_end = LEAST(slice_start + cur minutes, end)
        INSERT INTO TMP_TEL_HISTORY_RAW (...) SELECT :slice_no, :scope_id, ...
          FROM TABLE(DEV_TELEMETRY.INFORMATION_SCHEMA.TASK_HISTORY(
              SCHEDULED_TIME_RANGE_START => :slice_start, SCHEDULED_TIME_RANGE_END => :slice_end,
              RESULT_LIMIT => :limit, DATABASE_NAME => :db, SCHEMA_NAME => :sch))
        n = SQLROWCOUNT
        IF n >= limit AND cur > 1: DELETE that slice; limit_hits += 1; cur = GREATEST(1, FLOOR(cur/2))   -- re-read smaller
        ELSE: IF n >= limit THEN saturated = TRUE; rows += n; slice_start = slice_end; cur = slice_minutes
     ```

     *Why: passing DATABASE_NAME/SCHEMA_NAME to the function filters **before** the row limit, so other schemas can't use it up. Halving a full slice means a busy window is re-read completely instead of silently truncated.*
  4. Write a TMP_TEL_SCOPE_STATUS row: OK or SATURATED, plus rows read. SATURATED → partial = TRUE, with a warning recommending an ACCOUNT_USAGE backfill.
  5. **On exception:** delete that scope's staged rows, write a FAILED status row with the error, scopes_failed += 1, partial = TRUE, warning `SCOPE_FAILED`.
     *Why: one broken scope must not stop the others.*
- After the loop: no enabled scopes → warning. All scopes failed → raise a declared exception, so the run is FAILED.

**Step 4 — candidates**
- Count non-terminal staged rows (SCHEDULED/EXECUTING) as NONTERMINAL_RUNS_SEEN. They are not collected; a later run picks them up once they finish.
- `CREATE OR REPLACE TEMPORARY TABLE TMP_TEL_CANDIDATES AS`:
  - Join staged rows to their scope. Keep terminal states only, with `NAME ILIKE TASK_NAME_LIKE`.
  - `TASK_FQN = UPPER(COALESCE(r.DATABASE_NAME, s.DATABASE_NAME) || '.' || COALESCE(r.SCHEMA_NAME, s.SCHEMA_NAME) || '.' || r.NAME)`
  - `RV = FN_PARSE_RETURN_VALUE(RETURN_VALUE)`; `TASK_RUN_ID = FN_TASK_RUN_ID(...)`.
  - Output every execution-history column: GRAPH_NAME = NULL::VARCHAR(500) for now; DURATION_SECONDS and START_DELAY_SECONDS from DATEDIFF('millisecond')/1000, only when the times exist; counts from `RV:inserted` etc.; lineage from `RV:source` etc. (LEFT 500); RETURN_VALUE_STATUS = `RV:status`; ISSUES = `ARRAY_TO_STRING(RV:issues::ARRAY, ' ; ')` (LEFT 2000); CUSTOM_METRICS = `RV:metrics::OBJECT`; LOADER_EXECUTION_ID = `RV:execution_id`.
  - `QUALIFY ROW_NUMBER() OVER (PARTITION BY TASK_RUN_ID ORDER BY COMPLETED_TIME DESC NULLS LAST, SCOPE_ID) = 1 AND (QUERY_ID IS NULL OR ROW_NUMBER() OVER (PARTITION BY QUERY_ID ORDER BY SCHEDULED_TIME, TASK_RUN_ID) = 1)`.
    *Why: slice boundaries can repeat a row, and a cloud-services failover can list one execution twice. One QUERY_ID means one run.*
- Count the candidates (TERMINAL_RUNS_SEEN) and their minimum SCHEDULED_TIME.

**Step 5 — enrichment.** A failure here is never fatal.
- `UPDATE TMP_TEL_CANDIDATES c SET GRAPH_NAME = k.TASK_NAME FROM TASK_CATALOG k WHERE k.TASK_ID = c.ROOT_TASK_ID`.
- In an exception block: fill missing SOURCE/STREAM/TARGET from `V_PIPELINE_OBJECTS` (`UPPER(LOAD_TASK_NAME) = TASK_NAME`, enabled). The loader's own value wins; the registry only fills gaps. A failure becomes warning `LINEAGE_ENRICHMENT_FAILED`.

**Step 6 — atomic write**, one transaction:
1. `BEGIN TRANSACTION`.
2. UPDATE the lock row; this is the mutex.
3. `INSERT INTO TASK_EXECUTION_HISTORY (...) SELECT ... FROM TMP_TEL_CANDIDATES c WHERE NOT EXISTS (same TASK_RUN_ID, with h.SCHEDULED_TIME >= min_sched − 1 day) AND NOT EXISTS (same QUERY_ID, same pruning)`. Stamp COLLECTED_TIMESTAMP = v_collected_ts (taken once, just before the transaction), COLLECTOR_RUN_ID and HISTORY_SOURCE. `v_inserted = SQLROWCOUNT`.
   *Why the pruning predicate: a run never changes its scheduled time, so the anti-join only needs recent micro-partitions.*
4. `INSERT INTO TASK_METRIC_HISTORY (...) SELECT ... FROM VW_TASK_METRIC_DERIVATION d WHERE d.COLLECTOR_RUN_ID = :v_run_id AND d.COLLECTED_TIMESTAMP = :v_collected_ts`. These are metrics for exactly the rows just inserted; a transaction sees its own inserts.
5. **Advance watermarks** for scopes whose status is OK:

   ```
   WATERMARK_TS = IFF(WINDOW_START <= COALESCE(WATERMARK_TS, WINDOW_START),
                      GREATEST(COALESCE(WATERMARK_TS, WINDOW_END), WINDOW_END),
                      WATERMARK_TS)
   ```

   Also set LAST_SUCCESS_AT, LAST_COLLECTOR_RUN_ID, and clear LAST_ERROR.
   *Why: a watermark only moves when the window touched the old watermark. That keeps coverage contiguous, so a gap is never skipped.*
6. For scopes whose status is not OK, set LAST_ERROR and LAST_ERROR_AT.
7. `COMMIT`.

*Why one transaction: facts, metrics and watermarks commit together or not at all. There is never a run without its metrics, or a watermark past uncommitted data.*

**Step 7 — close and publish**
- Status = `PARTIAL` if partial, else `SUCCEEDED_WITH_WARNINGS` if there are warnings, else `SUCCEEDED`.
- Update the log row: ENDED_AT, MIN/MAX window from scope status, every counter, RUNS_ALREADY_COLLECTED = terminal − inserted, WARNINGS (` | `-separated, LEFT 16000).
- Build the result OBJECT (run id, mode, status, window, every counter) with SELECT INTO.
- **Publish the collector's own return value under contract v1:** `{"contract_version":1,"inserted":runs_inserted,"execution_id":run_id,"target":"...TASK_EXECUTION_HISTORY","metrics":{"metric_rows_inserted":..,"history_rows_read":..,"result_limit_hits":..,"collector_warnings":..}}` via `CALL SYSTEM$SET_RETURN_VALUE(:v_rv)`, in an exception block.
  *Why: the collector's own runs become ordinary telemetry.*
- RETURN the object.

**Procedure-level EXCEPTION handler:**
1. Capture `v_err = LEFT(SQLERRM, 4000)`.
2. `ROLLBACK`.
3. In a nested exception block: update the scopes' LAST_ERROR from TMP_TEL_SCOPE_STATUS. This block exists because the temp table may not exist yet.
4. Update the log row to FAILED, with counters, warnings and the error.
5. `RAISE`. *Why: the task run must show FAILED.*

Track warnings as a VARCHAR plus a count (not an ARRAY), so no semi-structured variable is ever bound.

### 3.7 `SP_REBUILD_TASK_METRICS(P_FROM TIMESTAMP_LTZ, P_TO TIMESTAMP_LTZ, P_METRIC_NAME VARCHAR DEFAULT NULL) RETURNS VARIANT`
1. Validate the window: from < to and ≤ 400 days, otherwise raise.
2. Log a `REBUILD_METRICS` run.
3. One transaction: take the lock; DELETE run metrics (`DIMENSION_KEY = 'RUN'`, optional metric filter) for runs with SCHEDULED_TIME in [from, to); re-INSERT from VW_TASK_METRIC_DERIVATION for the same runs, stamped with the rebuild id; COMMIT.
4. Update the log row; return the deleted and inserted counts.
5. EXCEPTION: rollback, log FAILED, raise.

*Why: lets you backfill a new metric or fix a rule using the same logic as the collector.*

### 3.8 `SP_APPLY_TELEMETRY_RETENTION() RETURNS VARIANT`
- Read the 4 retention settings, each with a floor of 30 days (*so a typo can't wipe history*).
- Delete, in this order:
  1. metric rows by OBSERVED_TIMESTAMP,
  2. execution rows by SCHEDULED_TIME,
  3. log rows by STARTED_AT (never RUNNING),
  4. catalog rows that are IS_DROPPED, by DROPPED_DETECTED_AT.
- Publish a contract-v1 return value with `"deleted": total`, and return the counts.

### 3.9 Views (readers get these through the future-views grant)

**`VW_COLLECTOR_HEALTH`** (one row, INCREMENTAL runs only)
- ENVIRONMENT and COLLECTOR_HEALTH:
  - `DOWN`: no successful run within COLLECTOR_STALE_ALERT_MINUTES
  - `DEGRADED`: the last run was FAILED, ABANDONED, PARTIAL or SUCCEEDED_WITH_WARNINGS
  - `HEALTHY`: otherwise
- LAST_SUCCESS_AT, MINUTES_SINCE_SUCCESS
- From the last run: id, status, started, runs inserted, metrics inserted, warnings, error
- Last 24 h: runs, failed, partial, runs inserted, metrics inserted, average seconds

**`VW_COLLECT_SCOPE_STATUS`**
- The scope columns, plus WATERMARK_LAG_MINUTES and the catalog/error fields.
- Tasks in catalog and suspended count: non-dropped, joined on pipeline + db + schema.
- Runs and failures in the last 24 h, by pipeline.

**`VW_TASK_LATEST_RUN`** (last 30 days, one row per task)
- The latest run by SCHEDULED_TIME desc, ATTEMPT desc: state, times, attempt, duration, counts, return-value status, error, MINUTES_SINCE_LAST_RUN.
- Last success time, last failure time, runs/failures/skips in the last 24 h.
- Current TASK_STATE, LAST_SUSPENDED_REASON, SCHEDULE and WAREHOUSE from the non-dropped catalog row.
- NOT_IN_CATALOG flag.

**`VW_TASK_RUN_METRICS`** (the reporting view)
- Execution facts LEFT JOIN a pivot of the run metrics: MAX(IFF(...)) for each core metric.
- CUSTOM_METRICS = `ARRAY_AGG(IFF(custom, OBJECT_CONSTRUCT('metric',..,'value',..,'unit',..), NULL))`, plus METRIC_ROWS.
- Don't use OBJECT_AGG; it errors on duplicate keys.

### 3.10 Tasks (created suspended)

**`TSK_TEL_COLLECT`**
- WAREHOUSE = DEV_TELEMETRY_WH
- SCHEDULE = `'USING CRON 10,25,40,55 7-19 * * 1-5 America/New_York'` (*5 minutes after each :05/:20/:35/:50 load cycle; PROD runs 24x7*)
- ALLOW_OVERLAPPING_EXECUTION = FALSE
- USER_TASK_TIMEOUT_MS = 900000
- SUSPEND_TASK_AFTER_NUM_FAILURES = 0 (*telemetry must never silently auto-suspend*)
- COMMENT
- TASK_AUTO_RETRY_ATTEMPTS = 1 (*safe because the collector is idempotent*)
- `AS CALL ...SP_COLLECT_TASK_TELEMETRY();`
- Keep the property order: warehouse, schedule, overlap, timeout, suspend, comment, auto-retry.

**`TSK_TEL_HOUSEKEEPING`**
- `'USING CRON 17 6 * * * America/New_York'`, USER_TASK_TIMEOUT_MS = 1800000, SUSPEND_TASK_AFTER_NUM_FAILURES = 0
- `AS CALL ...SP_APPLY_TELEMETRY_RETENTION();`

### 3.11 Smoke test
1. `SELECT FN_PARSE_RETURN_VALUE('{"inserted":100,"updated":15,"deleted":5,"metrics":{"rows_rejected":2}}')`.
2. `CALL SP_COLLECT_TASK_TELEMETRY()`. Expect SUCCEEDED (or SUCCEEDED_WITH_WARNINGS) and 0 runs inserted.
3. `SELECT` from the two health views.
4. Leave commented-out `ALTER TASK ... RESUME` lines, to run only after Stage 5 passes.

---

## STAGE 4 — `04_test_tasks.sql` (TEST ONLY, schema `TEL_TEST`)

*Why: the Virtuous graph doesn't exist yet. This graph creates every history shape the collector must handle. Its loaders do **real DML** and write a **ground-truth audit row**, so the tests can prove that telemetry counts equal what actually happened.*

**Start with:** `ALTER TASK IF EXISTS ...TSK_TEST_ROOT SUSPEND;`, then `DROP TASK IF EXISTS ...TSK_TEST_FINALIZER;` so a re-run can replace the root cleanly.

**Tables** (IF NOT EXISTS):
- `SEQ_TEST_ID`.
- `TEST_ORDERS` (ID NUMBER NOT NULL, AMOUNT NUMBER(12,2), STATUS VARCHAR(20), UPDATED_AT TIMESTAMP_LTZ, LAST_LOAD_ID VARCHAR(36)), and `TEST_CUSTOMERS` / `TEST_PAYMENTS` created `LIKE TEST_ORDERS`. Seed 500 rows each, only when empty.
- `TEST_EVENTS` (EVENT_ID, EVENT_TYPE, CREATED_AT) and `TEST_EVENTS_LANDED` (+ LANDED_AT).
- Stream `STR_TEST_EVENTS` ON TABLE TEST_EVENTS, APPEND_ONLY, created IF NOT EXISTS.
- `TEST_HEARTBEAT` (BEAT_AT, NOTE).
- `TEST_CONTROL` (TASK_SHORT_NAME, BEHAVIOR, PARAM_NUMBER, ENABLED, UPDATED_AT), PK (task, behavior).
- `TEST_LOAD_AUDIT` (LOADER_EXECUTION_ID, TASK_SHORT_NAME, ENTITY, STATUS, INSERTED, UPDATED, DELETED, RETURN_VALUE, ERROR_MESSAGE, GRAPH_RUN_GROUP_ID, STARTED_AT, ENDED_AT).
- `TEST_PIPELINE_REGISTRY` (PIPELINE_NAME, LOAD_TASK_NAME, SOURCE_OBJECT, STREAM_OBJECT, TARGET_OBJECT, ENABLED), seeded with MERGE for the 4 loaders:
  - ORDERS, CUSTOMERS and PAYMENTS: source `'TEL_TEST.SYNTHETIC_SOURCE'`, target = their table.
  - EVENTS: source TEST_EVENTS, stream STR_TEST_EVENTS, target TEST_EVENTS_LANDED.

**`SP_TEST_LOADER(P_ENTITY VARCHAR)`** — ORDERS, CUSTOMERS or PAYMENTS
1. Get the group id from `SYSTEM$TASK_RUNTIME_INFO('CURRENT_TASK_GRAPH_RUN_GROUP_ID')`, wrapped.
2. Read the TEST_CONTROL behaviours for `TSK_TEST_LOAD_<ENTITY>`:
   - `FAIL`: raise after partial DML
   - `SLOW n`: SYSTEM$WAIT n seconds first
   - `VOLUME x`: multiply the insert count
   - `RV_INVALID_JSON`: return value `'load finished OK'`
   - `RV_NEGATIVE`: `inserted = -5`
   - `RV_NONE`: no return value
3. Random volumes: inserts UNIFORM(50,200) × multiplier (capped at 20000), updates UNIFORM(5,30), deletes UNIFORM(0,8).
4. One transaction:
   - INSERT via `IDENTIFIER(:v_target)` from `GENERATOR(ROWCOUNT => 20000)` with `ROW_NUMBER() <= :n`. GENERATOR needs a constant, so filter afterwards. Stamp LAST_LOAD_ID = exec id.
   - If FAIL, raise the declared exception `'TEL_TEST injected failure: ...'`.
   - UPDATE n random rows **not touched by this run**; DELETE n random rows not touched by this run (`QUALIFY ROW_NUMBER() OVER (ORDER BY RANDOM()) <= :n`). Take every count from SQLROWCOUNT.
   - Build the contract-v1 JSON with metrics `{"rows_rejected":0,"dml_statements":3}`. **PAYMENTS omits source and target** (*this exercises the registry fallback*). Apply the RV behaviours.
   - INSERT the audit row with status SUCCEEDED; COMMIT.
5. Then `CALL SYSTEM$SET_RETURN_VALUE`, wrapped, and return the object.
6. EXCEPTION: rollback, insert a FAILED audit row with the error, RAISE.

**Other procedures:**
- **`SP_TEST_EVENTS_LOADER()`:** one transaction that inserts from the stream into TEST_EVENTS_LANDED (*this consumes the stream*), writes the audit row and commits. It sets a contract-v1 return value with source, stream and target.
- **`SP_TEST_HEARTBEAT()`:** inserts a heartbeat row and sets `{"contract_version":1,"inserted":1,"target":"...TEST_HEARTBEAT"}`.

**Tasks** (CREATE OR REPLACE, root first):

| Task | Definition | Exercises |
|---|---|---|
| TSK_TEST_ROOT | schedule `USING CRON 5,20,35,50 8-18 * * 1-5 America/New_York`, no overlap, timeout 600000, suspend-after 0; body: plain INSERT heartbeat | ABSENT return value |
| TSK_TEST_LOAD_ORDERS | AFTER root; `CALL SP_TEST_LOADER('ORDERS')` | PARSED + custom metrics |
| TSK_TEST_LOAD_CUSTOMERS | AFTER root | contract violations |
| TSK_TEST_LOAD_PAYMENTS | AFTER root | failure injection; registry lineage |
| TSK_TEST_LOAD_EVENTS | AFTER root, `WHEN SYSTEM$STREAM_HAS_DATA('...STR_TEST_EVENTS')`; calls the events loader | SKIPPED when empty |
| TSK_TEST_SQL_NO_RV | AFTER root; plain INSERT | ABSENT |
| TSK_TEST_SUMMARY | AFTER ORDERS, CUSTOMERS, PAYMENTS; plain INSERT | doesn't run when a predecessor fails |
| TSK_TEST_FINALIZER | `FINALIZE = ...TSK_TEST_ROOT`; `CALL SYSTEM$SET_RETURN_VALUE('{"contract_version":1,"metrics":{"finalizer_ran":1}}')` | NO_COUNTS; runs even after a failure |
| TSK_TEST_STANDALONE | own CRON schedule; `CALL SP_TEST_HEARTBEAT()` | graph of one |
| TSK_ADHOC_NOT_MONITORED | yearly CRON; plain INSERT | **must never be collected** (outside the scope pattern) |

**Helpers:**
- **`SP_TEST_SET_BEHAVIOR(task, behavior, enabled, param DEFAULT NULL)`:** validate the behaviour, then MERGE.
- **`SP_TEST_RESET_BEHAVIORS()`:** set every behaviour to disabled.
- **`SP_TEST_SET_MODE(mode)`:** always suspend the root first.
  - `OFF`: suspend everything.
  - Otherwise, resume all children and the finalizer.
  - `SCHEDULED`: also resume the root and the standalone task.
  - `MANUAL`: root and standalone stay suspended, so only EXECUTE TASK runs the graph. *That makes the tests deterministic.*
- **`SP_TEST_WAIT_FOR_GRAPH(group_id, attempt, timeout DEFAULT 600)`:** poll every 5 s (`SYSTEM$WAIT`). Finished = COMPLETE_TASK_GRAPHS(GRAPH_RUN_GROUP_ID => ...) has a row for that attempt **and** TASK_HISTORY(GRAPH_RUN_GROUP_ID => ...) has no SCHEDULED/EXECUTING rows (*so the finalizer is done*). On timeout, raise.
- **`SP_TEST_RUN_GRAPH(wait BOOLEAN DEFAULT TRUE, timeout DEFAULT 600)`:**
  1. `v_before = now`; `EXECUTE IMMEDIATE 'EXECUTE TASK ...TSK_TEST_ROOT'`.
  2. Poll TASK_HISTORY (TASK_NAME='TSK_TEST_ROOT', DATABASE_NAME, SCHEMA_NAME, start = v_before − 2 min) for up to 60 s, until a row with `SCHEDULED_FROM = 'EXECUTE_TASK'` and `SCHEDULED_TIME >= v_before − 5 s` gives the group id (MAX_BY).
  3. If wait is TRUE, call the wait procedure and read the final state from COMPLETE_TASK_GRAPHS.
  4. Return `{graph_run_group_id, attempt:1, state, first_error_task, requested_at}`.
- **`SP_TEST_RETRY_LAST(timeout)`:** find the latest completed group and attempt in COMPLETE_TASK_GRAPHS for the root (filtered to this db/schema); `EXECUTE IMMEDIATE 'EXECUTE TASK ... RETRY LAST'`; wait for attempt + 1 (compute it into a variable first); return the state.

**Wiring:**
1. MERGE the scope `('TEL_TEST', 'DEV_TELEMETRY', 'TEL_TEST', 'TSK_TEST%')`.
2. `CREATE OR REPLACE VIEW V_PIPELINE_OBJECTS` as a select from TEST_PIPELINE_REGISTRY.
3. `CALL SP_TEST_SET_MODE('MANUAL')`, then reset the behaviours.

**First look:** run the graph once, run the collector once, then show the last 20 TEL_TEST execution rows and the last 50 metric rows.

---

## STAGE 5 — `05_test_collector.sql` (test suite; run top to bottom in ONE session)

**Harness:**
- Table `TEL_TEST.TEST_RESULTS` (SUITE_RUN_ID, TEST_ID, DESCRIPTION, PASSED, DETAIL VARIANT, ERROR_MESSAGE, CHECKED_AT).
- Procedure `SP_TEST_ASSERT(P_TEST_ID, P_DESCRIPTION, P_SQL)`:
  - Reads `$SUITE_ID`, falling back to 'ADHOC'.
  - `EXECUTE IMMEDIATE :P_SQL`. **P_SQL must return one row with columns PASS (BOOLEAN) and DETAIL (VARIANT/OBJECT/NULL).**
  - Reads the result via RESULT_SCAN: `COALESCE(BOOLAND_AGG(COALESCE(PASS,FALSE)),FALSE)` and `MAX(TO_JSON(DETAIL::VARIANT))`. Zero rows, NULL or an error counts as FAIL.
  - Inserts the result, storing DETAIL with TRY_PARSE_JSON of the string.
  - Returns `'PASS  T..  desc'` or `'FAIL ...'`.
  - Session variables like `$G1` work inside P_SQL because the procedure runs with caller's rights.
- Inside assertions, wrap every BOOLAND_AGG condition in `COALESCE(cond, FALSE)`, because BOOLAND_AGG ignores NULLs.
- Pass assertion SQL as `$$ ... $$` string arguments.

**Session variables:**
- `SET SUITE_ID = (SELECT UUID_STRING())`; `SET SUITE_START = (SELECT CURRENT_TIMESTAMP())`.
- After each graph run or collection: `SET Gn = (SELECT $1:graph_run_group_id::VARCHAR FROM TABLE(RESULT_SCAN(LAST_QUERY_ID())))`, and similarly for collector run ids.

**Setup:** MANUAL mode, reset behaviours, run one collection so the suite starts from a caught-up state.

**The 37 assertions:**

**T01 — pure functions**
- **T01.1** The 16 parser cases from 3.2: status, ins, upd, del, rows_total (all via EQUAL_NULL), metric count = `ARRAY_SIZE(OBJECT_KEYS(p:metrics::OBJECT))` or 0, and issues present/absent. Put the non-null canonical row first in VALUES.
- **T01.2** Lineage, execution_id and contract_version are extracted, and rows_total = 0 for `{"contract_version":1,"inserted":0,"source":"S","stream":"ST","target":"T","execution_id":"abc"}`.
- **T01.3** The run key is deterministic, ignores name case, differs per attempt and per task, and is 32 characters on the fallback path.

**T02 — graph run 1 with PAYMENTS failing**
Setup: set FAIL on, `EXECUTE TASK TSK_ADHOC_NOT_MONITORED`, run the graph (G1), collect (C1).
- **T02.01** The terminal count in TASK_HISTORY(GRAPH_RUN_GROUP_ID => $G1) equals the collected count for G1, which equals the distinct keys (> 0).
- **T02.02** ORDERS and CUSTOMERS: SUCCEEDED, PARSED, and the counts equal TEST_LOAD_AUDIT (joined on LOADER_EXECUTION_ID); exactly 2 rows.
- **T02.03** At least 3 audit rows since the suite start carry GRAPH_RUN_GROUP_ID = $G1. This proves runtime info works in a procedure called by a task.
- **T02.04** PAYMENTS: FAILED, error ILIKE '%TEL_TEST injected failure%', ROWS_INSERTED NULL, attempt 1.
- **T02.05** PAYMENTS metrics: RUN_FAILED = 1 once, DURATION once, no ROWS% metrics.
- **T02.06** Lineage precedence: ORDERS source = 'TEL_TEST.GENERATOR' (from the loader); PAYMENTS source = 'TEL_TEST.SYNTHETIC_SOURCE' (from the registry), with the right targets.
- **T02.07** EVENTS: one run, SKIPPED, zero metric rows.
- **T02.08** ROOT and SQL_NO_RV: SUCCEEDED, ABSENT, 2 DURATION and 2 RUN_FAILED = 0 metrics, no ROWS%.
- **T02.09** Every G1 row has GRAPH_NAME = 'DEV_TELEMETRY.TEL_TEST.TSK_TEST_ROOT'.
- **T02.10** SUMMARY has no SUCCEEDED row in G1. Put the actual states in DETAIL.
- **T02.11** FINALIZER: SUCCEEDED, NO_COUNTS, and custom metric FINALIZER_RAN = 1 from RETURN_VALUE_CUSTOM.
- **T02.12** The ORDERS metric set is exactly `DML_STATEMENTS, DURATION_SECONDS, ROWS_DELETED, ROWS_INSERTED, ROWS_REJECTED, ROWS_TOTAL, ROWS_UPDATED, RUN_FAILED, START_DELAY_SECONDS` (LISTAGG ordered). ROWS_TOTAL = the sum, DURATION matches the fact row, RUN_FAILED = 0, the ROWS_REJECTED unit is ROWS (registered) and the DML_STATEMENTS unit is COUNT (unregistered).
- **T02.13** The ad-hoc task ran (TASK_HISTORY since the suite start shows ≥ 1) but has 0 execution rows and 0 catalog rows.
- **T02.14** C1's log row is SUCCEEDED*, RUNS_INSERTED > 0, and RUNS_INSERTED / METRICS_INSERTED equal the fact counts with COLLECTOR_RUN_ID = $C1.
- **T02.15** Catalog: ROOT and STANDALONE are IS_ROOT; FINALIZER is not root and FINALIZES_ROOT_TASK = the root FQN; SUMMARY has 3 predecessors; the 4 LOAD tasks are not roots.

**T03 — idempotency**
Save the TEL_TEST execution and metric counts, collect again, then collect in WINDOW mode over the last 6 h (C3).
- **T03.1** Both counts are unchanged.
- **T03.2** C3 is a WINDOW run, SUCCEEDED*, with RUNS_ALREADY_COLLECTED > 0.
- **T03.3** Zero duplicate TASK_RUN_IDs, zero duplicate QUERY_IDs, zero duplicate (run, metric, dimension), zero orphan metrics.

**T04 — RETRY LAST**
Save a HASH of PAYMENTS attempt 1 (key, state, error, collected_ts, collector id), set FAIL off, retry, collect.
- **T04.1** PAYMENTS in G1 has attempt 1 FAILED and attempt 2 SUCCEEDED with SCHEDULED_FROM 'MANUAL RETRY': 2 distinct keys.
- **T04.2** The attempt-1 hash is unchanged (append-only).
- **T04.3** Attempt-2 PAYMENTS counts equal the audit, and SUMMARY attempt 2 is SUCCEEDED.

**T05 — graph run 2: contract violations and stream data**
Setup: CUSTOMERS RV_INVALID_JSON, ORDERS RV_NEGATIVE, insert 25 events; run the graph (G2), reset, collect.
- **T05.1** CUSTOMERS: SUCCEEDED, INVALID_JSON, RETURN_VALUE = 'load finished OK', counts NULL, DURATION and RUN_FAILED = 0 present, no ROWS%.
- **T05.2** ORDERS: INVALID_VALUES, ROWS_INSERTED NULL, ROWS_UPDATED = audit, issues contain 'inserted must be a non-negative integer', no ROWS_TOTAL metric, a ROWS_UPDATED metric present.
- **T05.3** EVENTS: SUCCEEDED, ROWS_INSERTED = 25, stream lineage, and `NOT SYSTEM$STREAM_HAS_DATA(...)`.
- **T05.4** PAYMENTS: PARSED, with the registry target. SUMMARY: SUCCEEDED.

**T06 — in-flight run**
Setup: ORDERS SLOW 75; run the graph without waiting (G3); `SYSTEM$WAIT(25)`; collect (C6A).
- **T06.1** No ORDERS row for G3, and C6A's NONTERMINAL_RUNS_SEEN ≥ 1.

Then wait for G3, reset, collect.
- **T06.2** ORDERS for G3 is collected exactly once: SUCCEEDED, DURATION ≥ 75.

**T07 — outage**
Run the graph twice (G4, G5) with no collection in between, then collect once.
- **T07.1** For G4 and G5, the TASK_HISTORY terminal counts (since the suite start, NAME ILIKE 'TSK_TEST%') equal the collected counts. Use `COALESCE(eh.N, 0)` on the LEFT JOIN.
- **T07.2** The TEL_TEST WATERMARK_TS is ≥ the maximum END_TIME of G5, and LAST_ERROR is NULL.

**T08 — rebuild**
Save the HASH_AGG and count of TEL_TEST metric rows since the suite start (TASK_RUN_ID, METRIC_NAME, VALUE, UNIT, SOURCE, RUN_STATE). Run `SP_REBUILD_TASK_METRICS($SUITE_START, now)` (R8).
- **T08.1** The hash and count are identical, and R8's status is SUCCEEDED.

**T09 — dropped task**
Create TSK_TEST_TEMP_DROPME (standalone, yearly CRON, `SELECT 1`), refresh the catalog, save its id, DROP it, refresh again.
- **T09.1** The row with that id has IS_DROPPED and DROPPED_DETECTED_AT set.

**T10 — broken scope**
Insert scope `('TEST_BOGUS', 'DEV_TELEMETRY', 'not a valid name', '%')` and collect (C10).
- **T10.1** C10 is PARTIAL with SCOPES_FAILED = 1. The bogus scope's LAST_ERROR ILIKE '%plain UPPER identifiers%' and its WATERMARK_TS is NULL. The TEL_TEST scope's LAST_COLLECTOR_RUN_ID = $C10.

Delete the bogus scope afterwards.

**T11 — self-monitoring**
`EXECUTE TASK TSK_TEL_COLLECT`, `SYSTEM$WAIT(45)`, then collect.
- **T11.1** A log row since the suite start has INVOKED_BY_TASK ILIKE '%TSK_TEL_COLLECT%' and is SUCCEEDED*.
- **T11.2** TSK_TEL_COLLECT has an execution row since the suite start: SUCCEEDED, PARSED, with a METRIC_ROWS_INSERTED metric.

**Results:**
1. Reset behaviours; set MANUAL mode.
2. List the results for $SUITE_ID ordered by TEST_ID.
3. Show the summary row: tests, passed, failed, verdict `ALL TESTS PASSED` or `FAILURES`.
4. Show the run-by-run grid from VW_TASK_RUN_METRICS for TEL_TEST since the suite start.

---

## STAGE 6 — `06_operations.sql` (runbook; nothing scheduled)

| ID | Query | Purpose |
|---|---|---|
| O-1 | Health view + scope status view | Is the collector healthy? (DOWN outside the DEV schedule is expected) |
| O-2 | Last 20 collector runs, all counters + seconds | Recent runs |
| O-3 | VW_TASK_LATEST_RUN | Latest run per task |
| O-4 | FAILED/CANCELLED in the last 24 h with QUERY_ID and GRAPH_VERSION; groups with attempts > 1 (LISTAGG `attempt:state`) | Failures and retries |
| O-5 | Started, non-dropped catalog tasks with no run in 24 h; runs with NULL GRAPH_NAME | Coverage gaps |
| O-6 | Per task over 7 days of SUCCEEDED runs: % PARSED and counts per status, plus a sample issue | Contract compliance; go-live target 100% |
| O-7 | Duplicate run keys, duplicate query ids, duplicate (run, metric, dimension), orphan metrics, executed runs missing DURATION, PARSED successes missing ROWS_TOTAL, log rows stuck RUNNING > 30 min | Integrity: all must be 0 |
| O-8 | Session vars RECON_DB/RECON_SCHEMA; for one scope, TASK_HISTORY terminal runs in the last 24 h completed before the scope watermark vs collected rows; FULL OUTER JOIN per task with MISSING and OK/INVESTIGATE | Reconcile against Snowflake |
| O-9 | LAG(WINDOW_END) over successful INCREMENTAL runs; flag GAP when WINDOW_START > previous end | Window continuity |
| O-10 | ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY for DEV_TELEMETRY_WH, 30 days; collector seconds trend from the log | Cost |
| O-11 | Commented commands: WINDOW re-collect; ACCOUNT_USAGE backfill after > 7 days down; rebuild metrics (all or one); disable a scope; deactivate a custom metric | Recovery |
| O-12 | Commented steps (below) | Go-live onboarding |
| O-13 | Commented teardown: TEL_TEST only (repoint the view first) and full (admin) | Teardown |

**O-12 go-live steps:**
1. Grant USAGE on the Virtuous database and schema, plus `MONITOR ON ALL TASKS` and `MONITOR ON FUTURE TASKS IN SCHEMA`. *Without USAGE, names come back NULL; without MONITOR, SHOW TASKS hides the tasks.*
2. Insert the VIRTUOUS scope row.
3. Repoint V_PIPELINE_OBJECTS to the replication metadata table (placeholders).
4. Add the loader contract snippet: `SELECT OBJECT_CONSTRUCT(...)::VARCHAR INTO :v_rv; CALL SYSTEM$SET_RETURN_VALUE(:v_rv);`
5. Disable the TEL_TEST scope.

---

## 7. DEFINITION OF DONE
- [ ] Every file re-runs cleanly: no duplicates, no data loss, and no repoint undone.
- [ ] The Stage 2 schema guard returns `OK`.
- [ ] The Stage 3 smoke test returns SUCCEEDED (or SUCCEEDED_WITH_WARNINGS).
- [ ] Stage 5 verdict is `ALL TESTS PASSED` (37/37). Any FAIL comes with its DETAIL and your diagnosis.
- [ ] O-7 integrity checks are all 0, and O-8 reconciliation is all OK for TEL_TEST.
- [ ] No tunable is hard-coded, no credentials appear anywhere, every task was created suspended, and every name is fully qualified.
- [ ] Each stage ends with: the objects created, verification queries with expected results, and any risks.

## 8. KNOWN UNCERTAINTIES — verify, don't assume
Each of these is covered by a test:
1. Whether a JS UDF OBJECT keeps absent keys out (T01).
2. Bind variables in the TASK_HISTORY named arguments inside a procedure (smoke test).
3. SHOW + RESULT_SCAN inside MERGE USING in a caller's-rights procedure (T02.15).
4. The exact format of `task_relations.FinalizedRootTask` (T02.15).
5. Whether the finalizer row shares the root's ROOT_TASK_ID (T02.09).
6. SYSTEM$TASK_RUNTIME_INFO and SYSTEM$SET_RETURN_VALUE from a procedure called by a task (T02.03, T11).
7. Session variables visible to EXECUTE IMMEDIATE in a caller's-rights procedure (all T02).
8. RETRY LAST on a suspended root (T04).
9. What Snowflake records for a child whose predecessor failed (T02.10).

If any of these behaves differently, explain it and propose the smallest change that keeps the design intent.

**Start with STAGE 1 now.**
