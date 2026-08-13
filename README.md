Query 1 — Mary Beth's metadata mapping (you already have this)

Run it exactly as she sent it. This tells us which landing table maps to which satellite maps to which CURR view.

sql
select
    L.SRC_TABLE_NAME as SOURCE_LANDING_VIEW_NAME,
    L.DV_TABLE_NAME  as SATELLITE_TABLE_NAME,
    DV_TABLE_NAME || '_CURR' as BUSINESS_VAULT_CURR_VIEW
from
    (
        select distinct SRC_TABLE_NAME, DV_TABLE_NAME
        from PROD_DV.METADATA.VAULTSPEED_METADATA_EXPORT
        where source_name = 'INGENIUM'
          and SRC_PHYSICAL_SCHEMA = 'LANDING_ING'
        UNION
        select distinct SRC_TABLE_NAME || '__CT', DV_TABLE_NAME
        from PROD_DV.METADATA.VAULTSPEED_METADATA_EXPORT
        where source_name = 'INGENIUM'
          and SRC_PHYSICAL_SCHEMA = 'LANDING_ING'
    ) L
WHERE
    substr(DV_TABLE_NAME, 1, 2) = 'S_'
order by SRC_TABLE_NAME asc;

What I need from you: the full result set — how many rows, and the actual table name mappings.

Query 2 — Confirm the landing views actually exist

This checks what views are actually in PROD_DV.LANDING_ING right now, so we don't build queries against objects that don't exist.

sql
SELECT TABLE_NAME, TABLE_TYPE
FROM PROD_DV.INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'LANDING_ING'
ORDER BY TABLE_NAME;

What I need from you: total count and the full list.

Query 3 — Confirm the Raw Vault satellites actually exist

Same check, but for the satellite tables in Raw Vault.

sql
SELECT TABLE_NAME, TABLE_TYPE
FROM PROD_DV.INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'RAW_VAULT'
  AND TABLE_NAME LIKE 'S_%'
ORDER BY TABLE_NAME;

What I need from you: total count and the full list.

Query 4 — Confirm the Business Vault CURR views actually exist
sql
SELECT TABLE_NAME, TABLE_TYPE
FROM PROD_DV.INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'BUSINESS_VAULT'
  AND TABLE_NAME LIKE '%_CURR'
ORDER BY TABLE_NAME;

What I need from you: total count and the full list.

Query 5 — Check what columns each landing view has

We need to know which timestamp column exists per table for the dedup logic (remember — most tables use PREV_UPDT_TS, but some don't).

sql
SELECT TABLE_NAME, COLUMN_NAME
FROM PROD_DV.INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'LANDING_ING'
  AND COLUMN_NAME IN ('PREV_UPDT_TS', 'FILE_DATE', 'DV_LOAD_TIMESTAMP', 'LOAD_TIMESTAMP')
ORDER BY TABLE_NAME, COLUMN_NAME;

What I need from you: the full result — this tells me exactly which dedup timestamp to use per table, instead of guessing.

Query 6 — Check what columns the satellites have

Specifically looking for the timestamp and delete flag columns used in the CURR view logic.

sql
SELECT TABLE_NAME, COLUMN_NAME
FROM PROD_DV.INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'RAW_VAULT'
  AND TABLE_NAME LIKE 'S_%'
  AND COLUMN_NAME IN ('PREV_UPDT_TS', 'DV_LOAD_TIMESTAMP', 'DV_DELETE_FLAG', 'LOAD_DATE')
ORDER BY TABLE_NAME, COLUMN_NAME;

What I need from you: the full result.

Query 7 — Get the primary keys from Mary Beth's Excel (already in Snowflake?)

Check if someone already loaded the primary keys Excel file into a Snowflake table.

sql
SELECT TABLE_SCHEMA, TABLE_NAME
FROM PROD_DV.INFORMATION_SCHEMA.TABLES
WHERE TABLE_NAME ILIKE '%PRIMARY_KEY%'
   OR TABLE_NAME ILIKE '%PK%'
   OR TABLE_NAME ILIKE '%SRC_BK%'
ORDER BY TABLE_NAME;

If this returns nothing, also check your telemetry database:

sql
SELECT TABLE_SCHEMA, TABLE_NAME
FROM DEV_DATA_TELEMETRY.INFORMATION_SCHEMA.TABLES
WHERE TABLE_NAME ILIKE '%SRC_BK%'
   OR TABLE_NAME ILIKE '%PRIMARY%'
ORDER BY TABLE_NAME;

What I need from you: whether the PK reference table exists somewhere already, and if so, what it's called and what columns it has.

Query 8 — Sample one table end-to-end to confirm the pattern works

Pick TF_TABMC (the one you and Mary Beth tested live). Run these 3 counts:

8A — Landing dedup count:

sql
SELECT COUNT(*) AS landing_dedup_count
FROM (
    SELECT *,
           ROW_NUMBER() OVER (
             PARTITION BY CO_ID, AGT_ID, AGT_CNTRCT_EFF_DT,
                          AGT_COMM_YR, AGT_COMM_MO, AGT_COMM_PAYRL_DY
             ORDER BY PREV_UPDT_TS DESC
           ) AS rn
    FROM (
        SELECT * FROM PROD_DV.LANDING_ING.TF_TABMC
        UNION ALL
        SELECT * FROM PROD_DV.LANDING_ING.TF_TABMC__CT
    )
) WHERE rn = 1;

8B — Raw Vault satellite total count:

sql
SELECT COUNT(*) AS satellite_total_count
FROM PROD_DV.RAW_VAULT.S_ING_AGENT_GENERAL_AGENT_AGENTCOMMISSIONBALANCETABMC_LDS;

8C — Business Vault CURR count:

sql
SELECT COUNT(*) AS curr_view_count
FROM PROD_DV.BUSINESS_VAULT.S_ING_AGENT_GENERAL_AGENT_AGENTCOMMISSIONBALANCETABMC_LDS_CURR;

What I need from you: the 3 numbers. If they're close (within a few records), the pattern is confirmed and I can scale it to all tables.
