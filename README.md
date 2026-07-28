-- See all columns for a specific table
SELECT
    COLUMN_NAME,
    DATA_TYPE,
    IS_NULLABLE
FROM PROD_LANDING.INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'ING'
  AND TABLE_NAME   = 'TPOL'   -- change per table
ORDER BY ORDINAL_POSITION;

-- Test if POL_ID + CO_ID is truly unique in TPOL
SELECT
    COUNT(*)                              AS total_rows,
    COUNT(DISTINCT POL_ID || '|' || CO_ID) AS distinct_combos,
    CASE
        WHEN COUNT(*) = COUNT(DISTINCT POL_ID || '|' || CO_ID)
        THEN 'YES - this is a valid business key'
        ELSE 'NO - duplicates exist, wrong key'
    END AS is_valid_bk
FROM PROD_LANDING.ING.TPOL;
