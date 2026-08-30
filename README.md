"""
================================================================================
RAW VAULT RECONCILIATION  -  Landing  vs  Raw Vault Satellite
================================================================================

Implements the method Mary Beth specified:

    For every source table, take the primary/business keys from SRC_BK_REF,
    union the Landing base table with its __CT companion, deduplicate to one
    row per (primary key + applied date), and confirm every one of those
    versions exists in the Raw Vault satellite (Type 2, full history).

    A record missing with TODAY's applied date is acceptable - replication
    runs 10-30 minutes behind. A record missing with an OLDER applied date
    is a real defect and must be investigated.

Key difference from the earlier draft: the NOT EXISTS match is on
PRIMARY KEY *plus APPLIED DATE*, not primary key alone. Matching on the key
alone only proves "some version of this row exists" - it cannot detect the
failure mode where a reload leaves an old version in place and drops newer
ones (the SEI deployment incident). That is the whole reason this job exists.

Author: Lokesh Varma
================================================================================
"""

from snowflake.snowpark.context import get_active_session
from datetime import datetime
import uuid

session = get_active_session()

# ==============================================================================
# CONFIGURATION
# ==============================================================================

SRC_SYSTEM      = "ING"                  # ING | FRATDB | L70 | DI | LTC ...
LANDING_SCHEMA  = f"PROD_DV.LANDING_{SRC_SYSTEM}"
VAULT_SCHEMA    = "PROD_DV.RAW_VAULT"

BK_REF_TABLE    = "DEV_DATA_TELEMETRY.SRC_TGT_RECONCILIATION.SRC_BK_REF"
METADATA_TABLE  = "PROD_DV.METADATA.VAULTSPEED_METADATA_EXPORT"
RESULT_TABLE    = "DEV_DATA_TELEMETRY.SRC_TGT_RECONCILIATION.RECON_RESULTS"

# Applied-date column, per source system. There is no universal column name -
# this is the ordering column that decides which version of a row is newest.
APPLY_DATE_BY_SOURCE = {
    "ING":    ["PREV_UPDT_TS"],
    "FRATDB": ["PREV_UPDT_TS"],
    "L70":    ["CYCLE_DT"],
    "DI":     ["EXTF2_DATE"],
    "LTC":    ["FILE_DATE"],
    "DCLM":   ["FILE_DATE"],
    "SEI":    ["FILE_DATE"],
}

# Days of replication lag tolerated before a missing record counts as a defect.
LAG_TOLERANCE_DAYS = 1

# Cap concurrent async queries so we don't saturate the warehouse.
MAX_PARALLEL = 8

TEST_MODE  = True
TEST_LIMIT = 10

RUN_ID = str(uuid.uuid4())
RUN_TS = datetime.now()

# ==============================================================================
# RESULTS TABLE  (created once; the job APPENDS so history is preserved)
# ==============================================================================

session.sql(f"""
CREATE TABLE IF NOT EXISTS {RESULT_TABLE}
(
    RUN_ID              STRING,
    RUN_TS              TIMESTAMP_NTZ,
    SRC_SYSTEM          STRING,
    SOURCE_TABLE        STRING,
    TARGET_TABLE        STRING,
    BK_COLUMNS          STRING,
    APPLY_DATE_COLUMN   STRING,
    SOURCE_COUNT        NUMBER,   -- deduped landing versions (PK + apply date)
    TARGET_COUNT        NUMBER,   -- satellite rows, full history
    TARGET_KEY_COUNT    NUMBER,   -- distinct keys in satellite
    MISSING_TOTAL       NUMBER,
    MISSING_STALE       NUMBER,   -- older than tolerance -> real defect
    MISSING_IN_FLIGHT   NUMBER,   -- within tolerance -> replication lag
    OLDEST_MISSING_DT   TIMESTAMP_NTZ,
    STATUS              STRING,   -- PASS | WARN_IN_FLIGHT | FAIL | ERROR | SKIPPED
    ERROR_MESSAGE       STRING
)
""").collect()


# ==============================================================================
# HELPERS
# ==============================================================================

def columns_in(fq_schema: str, table: str) -> set:
    """Return the column names of one table, upper-cased."""
    db, schema = fq_schema.split(".", 1)
    rows = session.sql(f"""
        SELECT COLUMN_NAME
        FROM {db}.INFORMATION_SCHEMA.COLUMNS
        WHERE TABLE_SCHEMA = '{schema}'
          AND TABLE_NAME   = '{table}'
    """).collect()
    return {r["COLUMN_NAME"].upper() for r in rows}


def resolve_apply_date(src_table: str, tgt_table: str):
    """
    Pick the applied-date column that exists on BOTH the landing table and the
    satellite. Without it on both sides we cannot match a specific version, so
    the table is reported SKIPPED rather than silently checked key-only -
    a key-only check gives false confidence.
    """
    land_cols = columns_in(LANDING_SCHEMA, src_table)
    vault_cols = columns_in(VAULT_SCHEMA, tgt_table)
    for candidate in APPLY_DATE_BY_SOURCE.get(SRC_SYSTEM, []):
        if candidate in land_cols and candidate in vault_cols:
            return candidate, land_cols
    return None, land_cols


def build_landing_cte(src_table: str, bk_cols: str, apply_col: str,
                      land_cols: set) -> str:
    """
    Landing side: base UNION ALL __CT, BEFOREIMAGE excluded, deduplicated to
    one row per (primary key + applied date).

    BEFOREIMAGE rows are CDC bookkeeping emitted alongside every UPDATE - they
    are not business records and never land in the satellite. Leaving them in
    inflates the source count and produces phantom missing records on every
    update-heavy table.
    """
    op_filter = ("WHERE HEADER__OPERATION <> 'BEFOREIMAGE'"
                 if "HEADER__OPERATION" in land_cols else "")

    return f"""
    LANDING_ALL AS (
        SELECT {bk_cols}, {apply_col}
        FROM {LANDING_SCHEMA}.{src_table}

        UNION ALL

        SELECT {bk_cols}, {apply_col}
        FROM {LANDING_SCHEMA}.{src_table}__CT
        {op_filter}
    ),
    LANDING_DEDUP AS (
        SELECT {bk_cols}, {apply_col}
        FROM LANDING_ALL
        QUALIFY ROW_NUMBER() OVER (
            PARTITION BY {bk_cols}, {apply_col}
            ORDER BY {apply_col} DESC
        ) = 1
    )
    """


def build_recon_sql(src_table: str, tgt_table: str, bk_list: list,
                    apply_col: str, land_cols: set) -> str:
    """
    Match on PRIMARY KEY + APPLIED DATE, NULL-safe.

    EQUAL_NULL is used instead of '=' because a plain equality returns NULL
    when both sides are NULL, which NOT EXISTS reads as "no match" and reports
    as a missing record. Several FratDB tables have nullable key columns.
    """
    join_parts = [f"EQUAL_NULL(S.{c}, L.{c})" for c in bk_list]
    join_parts.append(f"EQUAL_NULL(S.{apply_col}, L.{apply_col})")
    join_condition = "\n              AND ".join(join_parts)

    bk_cols = ", ".join(bk_list)
    landing_cte = build_landing_cte(src_table, bk_cols, apply_col, land_cols)

    return f"""
    WITH {landing_cte},
    MISSING AS (
        SELECT L.{apply_col} AS APPLY_DT
        FROM LANDING_DEDUP L
        WHERE NOT EXISTS (
            SELECT 1
            FROM {VAULT_SCHEMA}.{tgt_table} S
            WHERE {join_condition}
        )
    )
    SELECT
        (SELECT COUNT(*) FROM LANDING_DEDUP)                        AS SOURCE_COUNT,
        (SELECT COUNT(*) FROM {VAULT_SCHEMA}.{tgt_table})           AS TARGET_COUNT,
        (SELECT COUNT(DISTINCT {bk_cols})
           FROM {VAULT_SCHEMA}.{tgt_table})                         AS TARGET_KEY_COUNT,
        (SELECT COUNT(*) FROM MISSING)                              AS MISSING_TOTAL,
        (SELECT COUNT(*) FROM MISSING
          WHERE APPLY_DT < DATEADD(DAY, -{LAG_TOLERANCE_DAYS}, CURRENT_DATE()))
                                                                    AS MISSING_STALE,
        (SELECT COUNT(*) FROM MISSING
          WHERE APPLY_DT >= DATEADD(DAY, -{LAG_TOLERANCE_DAYS}, CURRENT_DATE()))
                                                                    AS MISSING_IN_FLIGHT,
        (SELECT MIN(APPLY_DT) FROM MISSING)                         AS OLDEST_MISSING_DT
    """


# ==============================================================================
# METADATA  -  which source table maps to which satellite, and on what keys
# ==============================================================================

metadata_sql = f"""
SELECT DISTINCT
    B.SRC_TABLE_NAME,
    B.BK_COLUMN_VALUES,
    M.DV_TABLE_NAME
FROM {BK_REF_TABLE} B
JOIN {METADATA_TABLE} M
  ON UPPER(B.SRC_TABLE_NAME) = UPPER(M.SRC_TABLE_NAME)
WHERE B.SRC_SYSTEM_NAME = '{SRC_SYSTEM}'
  AND M.DV_TABLE_NAME LIKE 'S\\_%' ESCAPE '\\'
ORDER BY B.SRC_TABLE_NAME, M.DV_TABLE_NAME
"""

meta_rows = session.sql(metadata_sql).collect()
if TEST_MODE:
    meta_rows = meta_rows[:TEST_LIMIT]

print(f"Run {RUN_ID} - processing {len(meta_rows)} mappings for {SRC_SYSTEM}")


# ==============================================================================
# EXECUTE  -  submitted asynchronously in batches
# ==============================================================================
# The earlier version issued one blocking round trip per table. Across 200+
# tables that serial wait, not the SQL itself, is what made the run take
# 6-7 minutes. collect_nowait() submits queries and lets Snowflake run them
# concurrently.

results = []
pending = []


def drain(pending_batch):
    for item in pending_batch:
        row = {
            "RUN_ID": RUN_ID, "RUN_TS": RUN_TS, "SRC_SYSTEM": SRC_SYSTEM,
            "SOURCE_TABLE": item["src"], "TARGET_TABLE": item["tgt"],
            "BK_COLUMNS": item["bk"], "APPLY_DATE_COLUMN": item["apply_col"],
            "SOURCE_COUNT": None, "TARGET_COUNT": None, "TARGET_KEY_COUNT": None,
            "MISSING_TOTAL": None, "MISSING_STALE": None,
            "MISSING_IN_FLIGHT": None, "OLDEST_MISSING_DT": None,
            "STATUS": "ERROR", "ERROR_MESSAGE": None,
        }
        try:
            r = item["handle"].result()[0]
            stale = r["MISSING_STALE"]
            inflight = r["MISSING_IN_FLIGHT"]

            if stale > 0:
                status = "FAIL"
            elif inflight > 0:
                status = "WARN_IN_FLIGHT"
            else:
                status = "PASS"

            row.update({
                "SOURCE_COUNT": r["SOURCE_COUNT"],
                "TARGET_COUNT": r["TARGET_COUNT"],
                "TARGET_KEY_COUNT": r["TARGET_KEY_COUNT"],
                "MISSING_TOTAL": r["MISSING_TOTAL"],
                "MISSING_STALE": stale,
                "MISSING_IN_FLIGHT": inflight,
                "OLDEST_MISSING_DT": r["OLDEST_MISSING_DT"],
                "STATUS": status,
                "ERROR_MESSAGE": None,
            })
        except Exception as e:
            row["ERROR_MESSAGE"] = str(e)[:5000]

        results.append(row)
        flag = "" if row["STATUS"] in ("PASS",) else f"  <-- {row['STATUS']}"
        print(f"  {row['SOURCE_TABLE']} -> {row['TARGET_TABLE']}: "
              f"{row['STATUS']}{flag}")


for meta in meta_rows:
    src = meta["SRC_TABLE_NAME"]
    tgt = meta["DV_TABLE_NAME"]
    bk_raw = meta["BK_COLUMN_VALUES"]
    bk_list = [c.strip() for c in bk_raw.split(",") if c.strip()]

    try:
        apply_col, land_cols = resolve_apply_date(src, tgt)

        if apply_col is None:
            # No shared applied-date column. A key-only check here would pass
            # tables that are actually missing recent versions, so flag it for
            # manual review instead of reporting a misleading PASS.
            results.append({
                "RUN_ID": RUN_ID, "RUN_TS": RUN_TS, "SRC_SYSTEM": SRC_SYSTEM,
                "SOURCE_TABLE": src, "TARGET_TABLE": tgt, "BK_COLUMNS": bk_raw,
                "APPLY_DATE_COLUMN": None,
                "SOURCE_COUNT": None, "TARGET_COUNT": None,
                "TARGET_KEY_COUNT": None, "MISSING_TOTAL": None,
                "MISSING_STALE": None, "MISSING_IN_FLIGHT": None,
                "OLDEST_MISSING_DT": None,
                "STATUS": "SKIPPED",
                "ERROR_MESSAGE": "No applied-date column common to landing "
                                 "and satellite - version-level match not "
                                 "possible; needs manual mapping.",
            })
            print(f"  {src} -> {tgt}: SKIPPED (no applied-date column)")
            continue

        sql = build_recon_sql(src, tgt, bk_list, apply_col, land_cols)
        pending.append({
            "src": src, "tgt": tgt, "bk": bk_raw, "apply_col": apply_col,
            "handle": session.sql(sql).collect_nowait(),
        })

        if len(pending) >= MAX_PARALLEL:
            drain(pending)
            pending = []

    except Exception as e:
        results.append({
            "RUN_ID": RUN_ID, "RUN_TS": RUN_TS, "SRC_SYSTEM": SRC_SYSTEM,
            "SOURCE_TABLE": src, "TARGET_TABLE": tgt, "BK_COLUMNS": bk_raw,
            "APPLY_DATE_COLUMN": None,
            "SOURCE_COUNT": None, "TARGET_COUNT": None, "TARGET_KEY_COUNT": None,
            "MISSING_TOTAL": None, "MISSING_STALE": None,
            "MISSING_IN_FLIGHT": None, "OLDEST_MISSING_DT": None,
            "STATUS": "ERROR", "ERROR_MESSAGE": str(e)[:5000],
        })
        print(f"  {src} -> {tgt}: ERROR - {str(e)[:120]}")

drain(pending)


# ==============================================================================
# WRITE  -  append, never overwrite
# ==============================================================================
# Appending with a RUN_ID lets you answer "when did this table start failing",
# which an overwrite destroys.

if results:
    session.create_dataframe(results) \
           .write.mode("append").save_as_table(RESULT_TABLE)
    print(f"\n{len(results)} rows appended to {RESULT_TABLE}")
else:
    print("\nNo results generated")

summary = {}
for r in results:
    summary[r["STATUS"]] = summary.get(r["STATUS"], 0) + 1

print(f"\nRun {RUN_ID} summary:")
for status in ("PASS", "WARN_IN_FLIGHT", "FAIL", "SKIPPED", "ERROR"):
    if status in summary:
        print(f"  {status:16} {summary[status]}")

fails = [r for r in results if r["STATUS"] == "FAIL"]
if fails:
    print(f"\n{len(fails)} table(s) with stale missing records - investigate:")
    for r in sorted(fails, key=lambda x: -(x["MISSING_STALE"] or 0)):
        print(f"  {r['SOURCE_TABLE']:30} -> {r['TARGET_TABLE']:55} "
              f"{r['MISSING_STALE']} missing since {r['OLDEST_MISSING_DT']}")

print("\nCompleted")
