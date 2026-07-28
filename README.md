SELECT DISTINCT 
    SRC_TABLE_NAME,
    DV_TABLE_NAME,
    DV_COLUMN_TYPE
FROM dev_dv.metadata.vaultspeed_metadata_export
WHERE SRC_TABLE_NAME = 'TPOL'   -- change this to any table you want
ORDER BY DV_TABLE_NAME;


SELECT 
    SRC_TABLE_NAME,
    SRC_NAME_IN_BK,
    SRC_COLUMN_NAME,
    DV_COLUMN_TYPE
FROM dev_dv.metadata.vaultspeed_metadata_export
WHERE SRC_TABLE_NAME = 'TPOL'   -- change this
  AND DV_COLUMN_TYPE = 'BUSINESS_KEY';
SELECT
    m.SRC_TABLE_NAME,
    m.DV_TABLE_NAME,
    -- Generates the landing-side count query
    'SELECT COUNT(DISTINCT ' || 
        LISTAGG(m.SRC_COLUMN_NAME, ' || ''|'' || ') 
        WITHIN GROUP (ORDER BY m.SRC_COLUMN_NAME) ||
    ') AS landing_bk_count FROM PROD_LANDING.ING.' || 
    m.SRC_TABLE_NAME AS landing_query,
    -- Generates the Raw Vault hub-side count query  
    'SELECT COUNT(DISTINCT ' ||
        LISTAGG(m.SRC_NAME_IN_BK, ' || ''|'' || ')
        WITHIN GROUP (ORDER BY m.SRC_NAME_IN_BK) ||
    ') AS rv_bk_count FROM PRODDV.RAW_VAULT.' ||
    m.DV_TABLE_NAME || 
    ' WHERE BKCC = ''ING''' AS rawvault_query
FROM dev_dv.metadata.vaultspeed_metadata_export m
WHERE m.DV_COLUMN_TYPE = 'BUSINESS_KEY'
  AND m.DV_TABLE_NAME LIKE 'H_%'    -- hubs only
GROUP BY m.SRC_TABLE_NAME, m.DV_TABLE_NAME
ORDER BY m.SRC_TABLE_NAME;


-- Run these two together and compare the numbers
-- Landing side
SELECT 
    'PROD_LANDING.ING.TPOL'    AS source,
    COUNT(*)                    AS total_rows,
    COUNT(DISTINCT POL_ID || '|' || CO_ID) AS distinct_bk,
    MAX(PREV_UPDT_TS)           AS max_updated
FROM PROD_LANDING.ING.TPOL

UNION ALL

-- Raw Vault side (filter BKCC to your source)
SELECT
    'PRODDV.RAW_VAULT.H_POLICY' AS source,
    COUNT(*)                     AS total_rows,
    COUNT(DISTINCT POL_ID_BK || '|' || CO_ID_BK) AS distinct_bk,
    MAX(LOAD_DATE)               AS max_updated
FROM PRODDV.RAW_VAULT.H_POLICY
WHERE BKCC = 'ING';

-- Records in Raw Vault NOT in Landing (should be zero or explained)
SELECT POL_ID_BK, CO_ID_BK
FROM PRODDV.RAW_VAULT.H_POLICY
WHERE BKCC = 'ING'
EXCEPT
SELECT POL_ID, CO_ID
FROM PROD_LANDING.ING.TPOL;


INSERT INTO DATA_TELEMETRY.SRC_TGT_RECONCILIATION.SRC_TGT_COUNT
(
    SRC_SYSTEM_NAME,
    TABLE_NAME,
    SRC_RECORD_COUNT,
    SF_LNDG_RECORD_COUNT,
    SF_LNDG_BK_COUNT,
    SRC_LAST_UPDATED_TS,
    SF_LNDG_LAST_UPDATED_TS,
    SRC_MAX_CREATED_TS,
    SF_LNDG_MAX_CREATED_TS
)
SELECT
    'ING'                                              AS SRC_SYSTEM_NAME,
    'TPOL'                                             AS TABLE_NAME,
    src.src_row_count                                  AS SRC_RECORD_COUNT,
    lnd.lnd_row_count                                  AS SF_LNDG_RECORD_COUNT,
    lnd.lnd_bk_count                                   AS SF_LNDG_BK_COUNT,
    src.src_max_updt                                   AS SRC_LAST_UPDATED_TS,
    lnd.lnd_max_updt                                   AS SF_LNDG_LAST_UPDATED_TS,
    NULL                                               AS SRC_MAX_CREATED_TS,
    NULL                                               AS SF_LNDG_MAX_CREATED_TS
FROM
    -- Source side (Ingenium DB2)
    (SELECT 
        COUNT(*)          AS src_row_count,
        MAX(PREV_UPDT_TS) AS src_max_updt
     FROM KCINGPRC.TPOL WITH UR) src,
    -- Landing side (Snowflake)
    (SELECT
        COUNT(*)                                    AS lnd_row_count,
        COUNT(DISTINCT POL_ID || '|' || CO_ID)      AS lnd_bk_count,
        MAX(PREV_UPDT_TS)                           AS lnd_max_updt
     FROM PROD_LANDING.ING.TPOL) lnd;




