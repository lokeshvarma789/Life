-- =============================================================================
-- Q-QLIK -- Qlik CT Table Replication Validation (Step 2 in Siddesh's checklist)
-- =============================================================================
-- Purpose: confirm Qlik-loaded CT tables show a recent HEADER__TIMESTAMP,
-- consistent with Qlik Cloud's continuous CDC replication still running.
-- Scans all Qlik schemas (ING, FRATDB, RPLUS, FRAT_ING).
--
-- Validation rule per table:
--   FRESH         -> HEADER__TIMESTAMP within last 4 hours (CDC actively flowing)
--   STALE_TODAY   -> Loaded today, but most recent change is 4-24 hours old
--                    (CDC may have paused; check Qlik Cloud task status)
--   MISSED_TODAY  -> Most recent HEADER__TIMESTAMP is before today
--                    (replication is not running; investigate in Qlik Cloud UI)
--   NO_DATA       -> Table has no rows at all
--
-- NOTE: This validates replication *signal* from Snowflake's side only.
-- The authoritative completion time lives in Qlik Cloud and must be
-- cross-checked there for incident-grade confirmation.
-- =============================================================================

EXECUTE IMMEDIATE
$$
DECLARE
    sql_stmt STRING;
    rs RESULTSET;
BEGIN

    SELECT LISTAGG(
        'SELECT ''' || TABLE_SCHEMA || ''' AS SCHEMA_NAME, ' ||
        '''' || TABLE_NAME || ''' AS TABLE_NAME, ' ||
        '''QLIK CLOUD'' AS TOOL, ' ||
        'MAX(HEADER__TIMESTAMP) AS LAST_HEADER_TS, ' ||
        'DATEDIFF(HOUR, MAX(HEADER__TIMESTAMP), CURRENT_TIMESTAMP()) AS HOURS_SINCE_LAST, ' ||
        'CASE ' ||
        '  WHEN MAX(HEADER__TIMESTAMP) IS NULL ' ||
        '    THEN ''NO_DATA - table has no rows'' ' ||
        '  WHEN DATEDIFF(HOUR, MAX(HEADER__TIMESTAMP), CURRENT_TIMESTAMP()) <= 4 ' ||
        '    THEN ''FRESH - CDC actively flowing'' ' ||
        '  WHEN TO_DATE(MAX(HEADER__TIMESTAMP)) = CURRENT_DATE() ' ||
        '    THEN ''STALE_TODAY - check Qlik Cloud task'' ' ||
        '  ELSE ''MISSED_TODAY - investigate Qlik Cloud replication'' ' ||
        'END AS VALIDATION_STATUS ' ||
        'FROM PROD_LANDING.' || TABLE_SCHEMA || '.' || TABLE_NAME,
        ' UNION ALL '
    )
    INTO :sql_stmt
    FROM PROD_LANDING.INFORMATION_SCHEMA.TABLES
    WHERE TABLE_TYPE = 'BASE TABLE'
      AND TABLE_SCHEMA IN ('ING','FRATDB','RPLUS','FRAT_ING')
      AND TABLE_NAME LIKE '%__CT';

    sql_stmt := sql_stmt || ' ORDER BY HOURS_SINCE_LAST DESC NULLS FIRST';

    rs := (EXECUTE IMMEDIATE :sql_stmt);
    RETURN TABLE(rs);
END;
$$;


-- =============================================================================
-- Q-TALEND -- Talend CT Table Job Validation (Step 4 in Siddesh's checklist)
-- =============================================================================
-- Purpose: confirm Talend-loaded CT tables show today's HEADER__TIMESTAMP,
-- consistent with the scheduled Talend job having completed.
-- Scans all Talend schemas (L70, DCLM, SAP, DI, LTC, SEI, SF, SCI, REF, DMC).
--
-- Validation rule per table:
--   LOADED_TODAY  -> at least one row with TO_DATE(HEADER__TIMESTAMP) = today
--   LOADED_YESTERDAY -> most recent load is 1 day ago
--                       (acceptable for Mon mornings after weekend; otherwise
--                       check Talend TMC job status)
--   MISSED        -> most recent load is 2+ days ago and table is in a
--                    schema that normally loads daily
--   NO_DATA       -> Table has no rows at all
--
-- CAVEAT: Talend job completion time itself lives in Talend TMC, not Snowflake.
-- This query infers "did the job complete" from CT data arrival. For incident
-- confirmation, cross-check TMC for the job's actual completion status and any
-- error messages.
--
-- CAVEAT: L70 (Tuesday-Saturday cadence per the SOP) will legitimately show
-- LOADED_YESTERDAY on Mondays and MISSED on Tuesdays-after-long-weekend. This
-- query does not yet have day-of-week awareness -- treat L70 results as
-- advisory until that gets built into Q9's per-table sequence detection.
-- =============================================================================

EXECUTE IMMEDIATE
$$
DECLARE
    sql_stmt STRING;
    rs RESULTSET;
BEGIN

    SELECT LISTAGG(
        'SELECT ''' || TABLE_SCHEMA || ''' AS SCHEMA_NAME, ' ||
        '''' || TABLE_NAME || ''' AS TABLE_NAME, ' ||
        '''TALEND'' AS TOOL, ' ||
        'MAX(HEADER__TIMESTAMP) AS LAST_HEADER_TS, ' ||
        'DATEDIFF(DAY, TO_DATE(MAX(HEADER__TIMESTAMP)), CURRENT_DATE()) AS DAYS_SINCE_LAST, ' ||
        'CASE ' ||
        '  WHEN MAX(HEADER__TIMESTAMP) IS NULL ' ||
        '    THEN ''NO_DATA - table has no rows'' ' ||
        '  WHEN TO_DATE(MAX(HEADER__TIMESTAMP)) = CURRENT_DATE() ' ||
        '    THEN ''LOADED_TODAY - Talend job completed'' ' ||
        '  WHEN DATEDIFF(DAY, TO_DATE(MAX(HEADER__TIMESTAMP)), CURRENT_DATE()) = 1 ' ||
        '    THEN ''LOADED_YESTERDAY - check TMC if daily table'' ' ||
        '  ELSE ''MISSED - investigate Talend TMC job'' ' ||
        'END AS VALIDATION_STATUS ' ||
        'FROM PROD_LANDING.' || TABLE_SCHEMA || '.' || TABLE_NAME,
        ' UNION ALL '
    )
    INTO :sql_stmt
    FROM PROD_LANDING.INFORMATION_SCHEMA.TABLES
    WHERE TABLE_TYPE = 'BASE TABLE'
      AND TABLE_SCHEMA IN ('L70','DCLM','SAP','DI','LTC','SEI','SF','SCI','REF','DMC')
      AND TABLE_NAME LIKE '%__CT';

    sql_stmt := sql_stmt || ' ORDER BY DAYS_SINCE_LAST DESC NULLS FIRST';

    rs := (EXECUTE IMMEDIATE :sql_stmt);
    RETURN TABLE(rs);
END;
$$;
