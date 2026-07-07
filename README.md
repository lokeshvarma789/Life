
/* ============================================================================
   STEP 9 - VALIDATE THE DV_SOURCE_OBJECT_NAME -> TABLE NAME MAPPING
   ----------------------------------------------------------------------------
   WHAT THIS CHECKS:
     FMC_OBJECT_LOADING_HISTORY gives us DV_SOURCE_OBJECT_NAME (e.g.
     "PLAN_ILLUSTRATION") but we do not yet know if the matching CT table
     in PROD_LANDING is literally named PLAN_ILLUSTRATION__CT, or something
     else (different prefix, different case, extra qualifier, etc).

     This query pulls today's distinct object names straight from FMC, then
     checks the PROD_LANDING catalog for a table that matches the guessed
     pattern <NAME>__CT. If a match exists, MATCH_FOUND = 'Y' and we can
     trust the naming convention. If not, we see it immediately and know
     to ask Siddesh or search the catalog differently before building
     anything further on top of this guess.

   HOW TO USE:
     Run as-is. No edits needed - it pulls the object names dynamically
     from today's FMC activity rather than hardcoding the 5 examples
     we've seen so far.
============================================================================ */

USE ROLE SF_PROD_DATA_ENGINEER;
USE WAREHOUSE PROD_DATA_ENGINEER_WH;


WITH

/* Distinct objects FMC recorded activity for today, per schema. */
FMC_OBJECTS_TODAY AS (
    SELECT DISTINCT
        BKCC,
        DV_SOURCE_OBJECT_NAME,
        DV_SOURCE_OBJECT_NAME || '__CT'                                AS GUESSED_TABLE_NAME
    FROM PROD_DV.FMC.FMC_OBJECT_LOADING_HISTORY
    WHERE BKCC IN ('ING', 'FRAT_ING', 'FRATDB')
      AND DATE(LOAD_START_DATE) = CURRENT_DATE()
),

/* Every real CT table that actually exists in PROD_LANDING for these schemas. */
REAL_TABLES AS (
    SELECT
        TABLE_SCHEMA,
        TABLE_NAME
    FROM PROD_LANDING.INFORMATION_SCHEMA.TABLES
    WHERE TABLE_SCHEMA IN ('ING', 'FRAT_ING', 'FRATDB')
      AND TABLE_TYPE = 'BASE TABLE'
)

SELECT
    f.BKCC,
    f.DV_SOURCE_OBJECT_NAME,
    f.GUESSED_TABLE_NAME,
    CASE
        WHEN r.TABLE_NAME IS NOT NULL THEN 'Y'
        ELSE 'N'
    END                                                                AS MATCH_FOUND,
    r.TABLE_NAME                                                       AS ACTUAL_TABLE_NAME_IF_FOUND
FROM FMC_OBJECTS_TODAY f
LEFT JOIN REAL_TABLES r
    ON r.TABLE_SCHEMA = f.BKCC
   AND r.TABLE_NAME   = f.GUESSED_TABLE_NAME
ORDER BY MATCH_FOUND, f.BKCC, f.DV_SOURCE_OBJECT_NAME
;


/* ============================================================================
   HOW TO READ THE RESULTS

   MATCH_FOUND = 'Y'  -> the __CT guess is correct for this object, the
                         Supplementary Query 2 pattern from Step9_DAG_Monitoring
                         can be trusted for this table as-is.

   MATCH_FOUND = 'N'  -> the naming guess failed for this object. Next step
                         is to search the catalog more broadly, e.g.:

     SELECT TABLE_SCHEMA, TABLE_NAME
     FROM PROD_LANDING.INFORMATION_SCHEMA.TABLES
     WHERE TABLE_SCHEMA = '<BKCC from the failed row>'
       AND TABLE_NAME ILIKE '%<part of DV_SOURCE_OBJECT_NAME>%';

   If MOST rows show 'Y' and only a few show 'N', the convention is real but
   has known exceptions - safe to proceed, just handle exceptions manually.

   If MOST rows show 'N', the naming convention is different than guessed
   and the join logic in Supplementary Query 2 needs to be rebuilt entirely
   based on whatever pattern the 'N' search above reveals.
============================================================================ */
