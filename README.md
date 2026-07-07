/* ============================================================================
   STEP 9 - SECOND VALIDATION ATTEMPT
   ----------------------------------------------------------------------------
   FIRST GUESS FAILED COMPLETELY: DV_SOURCE_OBJECT_NAME + '__CT' matched
   ZERO of 125 real PROD_LANDING tables. That guess is dead - do not use it.

   NEW HYPOTHESIS:
     DV_SOURCE_OBJECT_NAME values (AGENT, AGENCY, BANK_ACCOUNT, BENEFICIARY)
     look like clean source-system entity names, not landing CT table codes.
     Per the VaultSpeed naming convention doc reviewed earlier, the STAGE
     layer is organized by source with schemas like ING_STG, ING_EXT,
     ING_DFV in PROD_DV - and staging layers typically preserve the
     ORIGINAL source table name before Data Vault hub/sat/link renaming.

     This query searches PROD_DV for any schema matching that pattern and
     checks whether a table named exactly like DV_SOURCE_OBJECT_NAME exists
     there - no guessing at a suffix this time, exact name match only.

   HOW TO USE:
     Run as-is. No edits needed.
============================================================================ */

USE ROLE SF_PROD_DATA_ENGINEER;
USE WAREHOUSE PROD_DATA_ENGINEER_WH;


WITH

/* Distinct objects FMC recorded activity for today. */
FMC_OBJECTS_TODAY AS (
    SELECT DISTINCT
        BKCC,
        DV_SOURCE_OBJECT_NAME
    FROM PROD_DV.FMC.FMC_OBJECT_LOADING_HISTORY
    WHERE BKCC IN ('ING', 'FRAT_ING', 'FRATDB')
      AND DATE(LOAD_START_DATE) = CURRENT_DATE()
),

/* Every table across ALL schemas in PROD_DV - cast a wide net since we
   don't yet know the exact schema name pattern (ING_STG? ING_EXT? something
   else entirely). Exact name match against DV_SOURCE_OBJECT_NAME, no
   suffix guessing this time. */
ALL_PROD_DV_TABLES AS (
    SELECT
        TABLE_SCHEMA,
        TABLE_NAME
    FROM PROD_DV.INFORMATION_SCHEMA.TABLES
    WHERE TABLE_TYPE = 'BASE TABLE'
)

SELECT
    f.BKCC,
    f.DV_SOURCE_OBJECT_NAME,
    t.TABLE_SCHEMA                                                     AS FOUND_IN_SCHEMA,
    t.TABLE_NAME                                                       AS FOUND_TABLE_NAME
FROM FMC_OBJECTS_TODAY f
LEFT JOIN ALL_PROD_DV_TABLES t
    ON t.TABLE_NAME = f.DV_SOURCE_OBJECT_NAME
ORDER BY
    CASE WHEN t.TABLE_NAME IS NULL THEN 1 ELSE 0 END,
    f.BKCC, f.DV_SOURCE_OBJECT_NAME
;


/* ============================================================================
   HOW TO READ THE RESULTS

   FOUND_IN_SCHEMA populated  -> exact match exists somewhere in PROD_DV.
                                 Note which schema(s) it shows up in - if it's
                                 consistently one schema per BKCC (e.g. always
                                 "ING_STG" for BKCC='ING'), that confirms the
                                 pattern and we can build the join properly.

   FOUND_IN_SCHEMA is NULL    -> no exact match anywhere in PROD_DV either.
                                 If this happens for most/all rows, the object
                                 name likely lives in a database we haven't
                                 checked at all (maybe still in PROD_LANDING
                                 under a totally different naming scheme, or
                                 in a source-side system we don't have direct
                                 access to). At that point this is a question
                                 for Siddesh rather than something to keep
                                 guessing at - worth asking him directly:
                                 "what does DV_SOURCE_OBJECT_NAME map to?"
============================================================================ */
