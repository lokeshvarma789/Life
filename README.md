Build a Raw Vault reconciliation script in Snowflake (Snowpark Python) that verifies
every version of a source record actually landed in the Data Vault, not just that
the record's key exists somewhere.

BACKGROUND / WHY THIS EXISTS:
A prior incident showed that a table reload can leave an old version of a record
"current" while silently dropping newer versions — a simple "does this key exist"
check would report that as PASS. The script must catch missing *versions*, which
means matching on primary key PLUS the applied-date/timestamp column, not the key
alone.

SOURCE OF TRUTH TABLES:
- Business/primary keys per source table: DEV_DATA_TELEMETRY.SRC_TGT_RECONCILIATION.SRC_BK_REF
  (columns: SRC_SYSTEM_NAME, SRC_TABLE_NAME, BK_COLUMN_VALUES — comma-separated key list)
- Source-table-to-satellite mapping: PROD_DV.METADATA.VAULTSPEED_METADATA_EXPORT
  (columns: SRC_TABLE_NAME, DV_TABLE_NAME — satellite tables are prefixed "S_")
- Results table (append-only, not overwrite): DEV_DATA_TELEMETRY.SRC_TGT_RECONCILIATION.RECON_RESULTS

DATA FLOW TO RECONCILE:
- Source (base):      PROD_DV.LANDING_<SRC_SYSTEM>.<table>
- Source (CDC/delta):  PROD_DV.LANDING_<SRC_SYSTEM>.<table>__CT
- Target (Type 2 history, full record versions): PROD_DV.RAW_VAULT.<satellite>
- Do NOT compare against the _CURR view — that's Type 1/current-only and will
  always look "wrong" against history by design.

LOGIC REQUIRED:
1. For each source table in SRC_BK_REF (filterable by source system, e.g. 'ING',
   'FRATDB', 'L70'), look up its matching satellite(s) via VAULTSPEED_METADATA_EXPORT.
2. UNION ALL the base table with its __CT companion.
3. EXCLUDE rows where HEADER__OPERATION = 'BEFOREIMAGE' — these are CDC bookkeeping
   artifacts, not real business records, and will never appear in the satellite.
4. Deduplicate the unioned set to one row per (primary key columns + applied-date
   column) using QUALIFY ROW_NUMBER() OVER (PARTITION BY key, apply_date ORDER BY
   apply_date DESC) = 1.
5. The applied-date column is NOT the same name across sources — resolve it
   dynamically per source system:
     ING / FRATDB -> PREV_UPDT_TS
     L70          -> CYCLE_DT
     DI           -> EXTF2_DATE
     LTC / DCLM   -> FILE_DATE
   If neither the landing table nor the satellite has a shared applied-date
   column, do NOT fall back to a key-only match — mark the table SKIPPED and
   flag it for manual review, since a key-only check would produce false
   confidence.
6. For each (primary key + applied date) combination in the deduplicated source
   set, check whether it exists in the target satellite using NOT EXISTS —
   match every key column plus the applied-date column, using EQUAL_NULL (not
   plain =) since some key columns are nullable and plain equality treats
   NULL=NULL as no-match.
7. Classify each missing record by age:
   - Missing with today's applied date -> acceptable replication lag (WARN)
   - Missing with an older applied date -> real defect (FAIL)
   Use a configurable lag-tolerance-in-days variable (default 1 day).

OUTPUT PER TABLE:
SOURCE_TABLE, TARGET_TABLE, BK_COLUMNS, APPLY_DATE_COLUMN, SOURCE_COUNT (deduped),
TARGET_COUNT (full history), TARGET_KEY_COUNT (distinct keys), MISSING_TOTAL,
MISSING_STALE, MISSING_IN_FLIGHT, OLDEST_MISSING_DATE, STATUS (PASS / WARN_IN_FLIGHT
/ FAIL / SKIPPED / ERROR), ERROR_MESSAGE.

PERFORMANCE:
This runs across 200+ tables — do not issue one blocking query per table in a
serial loop (that's what makes it slow, not the SQL). Use Snowpark's
collect_nowait() to submit queries in parallel batches (configurable batch size,
e.g. 8 at a time) and drain results as they complete.

TESTING / OUTPUT MODES (must be switchable via one config variable):
- "preview"   -> print results only, write nothing (for first-run validation)
- "temp"      -> write to a session-scoped TEMPORARY table (safe to test with,
                 disappears automatically, no schema clutter)
- "permanent" -> APPEND (never overwrite) to RECON_RESULTS with a RUN_ID and
                 RUN_TS, so failure history can be trended over time

Also include a TABLE_FILTER list (empty = run everything) so a handful of named
tables can be tested before scaling to the full source system — always include
at least one table with a manually-verified expected result as a control case.

Error-handle per table (one bad table must not kill the whole run) — capture
the exception message and continue.

Print a run summary at the end: count by STATUS, and a sorted list of FAIL
tables with their oldest missing date, so the worst offenders are visible
immediately without querying the results table.
