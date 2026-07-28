-- Count of distinct ING tables with confirmed business keys
SELECT
    SRC_TABLE_NAME,
    LISTAGG(SRC_COLUMN_NAME, ', ')
        WITHIN GROUP (ORDER BY SRC_COLUMN_NAME) AS BUSINESS_KEYS,
    COUNT(*) AS BK_COLUMN_COUNT
FROM dev_dv.metadata.vaultspeed_metadata_export
WHERE DV_COLUMN_TYPE = 'BUSINESS_KEY'
  AND SRC_NAME_IN_BK = 'ING'
GROUP BY SRC_TABLE_NAME
ORDER BY SRC_TABLE_NAME;


-- Categorize the 210 gap tables
SELECT
    t.TABLE_NAME,
    CASE
        WHEN t.TABLE_NAME LIKE '%_CT'
            THEN 'CT table (change tracking - skip, not a base table)'
        WHEN t.TABLE_NAME LIKE 'TF_%'
            THEN 'TF_ prefix (Talend flat file staging - may not need BK)'
        ELSE 'Needs manual BK investigation'
    END AS CATEGORY,
    t.TABLE_ROWS
FROM PROD_LANDING.INFORMATION_SCHEMA.TABLES t
WHERE t.TABLE_SCHEMA = 'ING'
  AND t.TABLE_NAME NOT IN (
      SELECT DISTINCT SRC_TABLE_NAME
      FROM dev_dv.metadata.vaultspeed_metadata_export
      WHERE SRC_NAME_IN_BK = 'ING'
        AND DV_COLUMN_TYPE = 'BUSINESS_KEY'
  )
ORDER BY CATEGORY, t.TABLE_NAME;

-- Preview the data that would go into SRC_BK_REF
-- Do NOT run INSERT yet - just SELECT to verify
SELECT
    SRC_NAME_IN_BK      AS SRC_SYSTEM_NAME,
    SRC_TABLE_NAME      AS SRC_TABLE_NAME,
    SRC_COLUMN_NAME     AS BK_COLUMN_VALUES
FROM dev_dv.metadata.vaultspeed_metadata_export
WHERE DV_COLUMN_TYPE = 'BUSINESS_KEY'
  AND SRC_NAME_IN_BK  = 'ING'
ORDER BY SRC_TABLE_NAME, SRC_COLUMN_NAME;
