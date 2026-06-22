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
-- Empty tables (no rows) are excluded via HAVING MAX(HEADER__TIMESTAMP) IS NOT NULL.
--
-- COLUMNS:
--   HOURS_SINCE_LAST -- hours since last change, computed against SYSDATE() (UTC)
--                       to align with HEADER__TIMESTAMP, which Qlik writes as
--                       TIMESTAMP_NTZ assumed UTC. Using CURRENT_TIMESTAMP()
--                       (account timezone) produced negative values when the
--                       account TZ differs from UTC.
--   DAYS_STALE       -- whole days between last load and today; same semantics as Q1
--                       so values can be compared directly against the severity table.
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
        'DATEDIFF(HOUR, MAX(HEADER__TIMESTAMP), SYSDATE()) AS HOURS_SINCE_LAST, ' ||
        'DATEDIFF(DAY, MAX(HEADER__TIMESTAMP), CURRENT_DATE()) AS DAYS_STALE, ' ||
        'CASE ' ||
        '  WHEN DATEDIFF(HOUR, MAX(HEADER__TIMESTAMP), SYSDATE()) <= 4 ' ||
        '    THEN ''FRESH - CDC actively flowing'' ' ||
        '  WHEN TO_DATE(MAX(HEADER__TIMESTAMP)) = CURRENT_DATE() ' ||
        '    THEN ''STALE_TODAY - check Qlik Cloud task'' ' ||
        '  ELSE ''MISSED_TODAY - investigate Qlik Cloud replication'' ' ||
        'END AS VALIDATION_STATUS ' ||
        'FROM PROD_LANDING.' || TABLE_SCHEMA || '.' || TABLE_NAME || ' ' ||
        'HAVING MAX(HEADER__TIMESTAMP) IS NOT NULL',
        ' UNION ALL '
    )
    INTO :sql_stmt
    FROM PROD_LANDING.INFORMATION_SCHEMA.TABLES
    WHERE TABLE_TYPE = 'BASE TABLE'
      AND TABLE_SCHEMA IN ('ING','FRATDB','RPLUS','FRAT_ING')
      AND TABLE_NAME LIKE '%__CT';

    sql_stmt := sql_stmt || ' ORDER BY DAYS_STALE DESC, HOURS_SINCE_LAST DESC';

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
--   LOADED_TODAY     -> at least one row with TO_DATE(HEADER__TIMESTAMP) = today
--   LOADED_YESTERDAY -> most recent load is 1 day ago
--                       (acceptable for Mon mornings after weekend; otherwise
--                       check Talend TMC job status)
--   MISSED           -> most recent load is 2+ days ago
--                       (check Talend TMC job for errors)
--   EXPECTED_GAP     -> L70-only: today is Sunday or Monday and last load was
--                       Saturday (within L70's documented Tue-Sat schedule).
--                       Do NOT investigate; this is normal.
-- Empty tables (no rows) are excluded via HAVING MAX(HEADER__TIMESTAMP) IS NOT NULL.
--
-- L70 SPECIAL HANDLING:
-- Per the SOP source-schema table, L70 loads Tuesday-Saturday only. So on
-- Sundays the last expected load was Saturday (1 day ago), and on Mondays it
-- was Saturday (2 days ago). Without this carve-out, every L70 table would
-- false-flag on Sun/Mon. The DAYOFWEEK check (0=Sun, 1=Mon) handles this
-- using L70's documented cadence. All other Talend schemas use the normal
-- daily logic.
--
-- CAVEAT: Talend job completion time itself lives in Talend TMC, not Snowflake.
-- This query infers "did the job complete" from CT data arrival. For incident
-- confirmation, cross-check TMC for the job's actual completion status and any
-- error messages.
--
-- CAVEAT: LTC ("varies, may be weekly" per the SOP) is still treated as daily
-- here -- LTC tables may show LOADED_YESTERDAY or MISSED legitimately. Confirm
-- with team before raising LTC findings as incidents.
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
        -- L70 Tue-Sat carve-out (DAYOFWEEK: 0=Sun, 1=Mon, ..., 6=Sat)
        -- L70 is documented Tue-Sat per the SOP. Sun/Mon gaps are expected,
        -- so do not flag them as missed.
        '  WHEN ''' || TABLE_SCHEMA || ''' = ''L70'' ' ||
        '   AND DAYOFWEEK(CURRENT_DATE()) = 0 ' ||  -- Sunday
        '   AND DATEDIFF(DAY, TO_DATE(MAX(HEADER__TIMESTAMP)), CURRENT_DATE()) <= 1 ' ||
        '    THEN ''EXPECTED_GAP - L70 Sunday (Tue-Sat schedule)'' ' ||
        '  WHEN ''' || TABLE_SCHEMA || ''' = ''L70'' ' ||
        '   AND DAYOFWEEK(CURRENT_DATE()) = 1 ' ||  -- Monday
        '   AND DATEDIFF(DAY, TO_DATE(MAX(HEADER__TIMESTAMP)), CURRENT_DATE()) <= 2 ' ||
        '    THEN ''EXPECTED_GAP - L70 Monday (Tue-Sat schedule)'' ' ||
        -- Normal rules apply to all other Talend schemas, and to L70 Tue-Sat
        '  WHEN TO_DATE(MAX(HEADER__TIMESTAMP)) = CURRENT_DATE() ' ||
        '    THEN ''LOADED_TODAY - Talend job completed'' ' ||
        '  WHEN DATEDIFF(DAY, TO_DATE(MAX(HEADER__TIMESTAMP)), CURRENT_DATE()) = 1 ' ||
        '    THEN ''LOADED_YESTERDAY - check TMC if daily table'' ' ||
        '  ELSE ''MISSED - investigate Talend TMC job'' ' ||
        'END AS VALIDATION_STATUS ' ||
        'FROM PROD_LANDING.' || TABLE_SCHEMA || '.' || TABLE_NAME || ' ' ||
        'HAVING MAX(HEADER__TIMESTAMP) IS NOT NULL',
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
