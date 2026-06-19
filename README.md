# Life
-- ══════════════════════════════════════════════════════════════════════
-- NEW QUERIES — built to close the gap in Siddesh's "Required Snowflake
-- Queries" list. These are ADDITIONS to the existing SNOWFLAKE_MONITORING_
-- QUERIES.docx (Q0–Q6). Proposed numbering: Q7, Q8, Q9.
-- All columns used are confirmed: HEADER__OPERATION, HEADER__TIMESTAMP
-- (CT tables) and DAG_NAME, LOAD_START_DATE, LOAD_END_DATE, SUCCESS_FLAG
-- (FMC_LOAD_BATCH_HISTORY). No invented columns.
-- ══════════════════════════════════════════════════════════════════════

USE ROLE SF_PROD_DATA_ENGINEER;
USE DATABASE PROD_LANDING;
USE WAREHOUSE PROD_DATA_ENGINEER_WH;


-- ──────────────────────────────────────────────────────────────────────
-- Q7 — Daily Volume + Frequency Validation: SD AND IQR (Siddesh Step 5)
-- ──────────────────────────────────────────────────────────────────────
-- Gap being closed: existing Q1 already classifies LOAD_PATTERN
-- (DAILY/WEEKLY/MONTHLY/IRREGULAR) using simple count-based rules, but it
-- does NOT compute standard deviation or IQR on today's insert/update/
-- delete volume. Siddesh's Step 5 explicitly asks for BOTH methods,
-- clearly differentiated, per table.
--
-- Approach: for each CT table, build a 30-day daily history of total
-- operations (insert+update+delete), then compare TODAY's volume against
-- that history using both SD (mean +/- 2-3 stddev) and IQR (Q1/Q3 +/-
-- 1.5*IQR, the standard Tukey fence). Change schema/table per run, same
-- pattern as existing Q2.

WITH daily_history AS (
    SELECT
        TO_DATE(HEADER__TIMESTAMP)                                 AS LOAD_DATE,
        COUNT(CASE WHEN HEADER__OPERATION = 'INSERT' THEN 1 END)
          + COUNT(CASE WHEN HEADER__OPERATION = 'UPDATE' THEN 1 END)
          + COUNT(CASE WHEN HEADER__OPERATION = 'DELETE' THEN 1 END)  AS TOTAL_OPS
    FROM PROD_LANDING.L70.BTRLR__CT          -- CHANGE SCHEMA AND TABLE NAME
    WHERE TO_DATE(HEADER__TIMESTAMP) >= DATEADD(day, -30, CURRENT_DATE())
      AND TO_DATE(HEADER__TIMESTAMP) <  CURRENT_DATE()              -- history excludes today
    GROUP BY 1
),
stats AS (
    SELECT
        COUNT(*)                                                   AS SAMPLE_DAYS,
        AVG(TOTAL_OPS)                                             AS MEAN_OPS,
        STDDEV(TOTAL_OPS)                                          AS STDDEV_OPS,
        PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY TOTAL_OPS)     AS Q1_OPS,
        PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY TOTAL_OPS)     AS Q3_OPS
    FROM daily_history
),
today AS (
    SELECT
        COUNT(CASE WHEN HEADER__OPERATION = 'INSERT' THEN 1 END)
          + COUNT(CASE WHEN HEADER__OPERATION = 'UPDATE' THEN 1 END)
          + COUNT(CASE WHEN HEADER__OPERATION = 'DELETE' THEN 1 END)  AS TODAY_OPS
    FROM PROD_LANDING.L70.BTRLR__CT          -- SAME TABLE AS ABOVE
    WHERE TO_DATE(HEADER__TIMESTAMP) = CURRENT_DATE()
)
SELECT
    t.TODAY_OPS,
    s.SAMPLE_DAYS,
    s.MEAN_OPS,
    s.STDDEV_OPS,
    s.MEAN_OPS - (2 * s.STDDEV_OPS)                                AS SD_LOWER_2,
    s.MEAN_OPS + (2 * s.STDDEV_OPS)                                AS SD_UPPER_2,
    s.MEAN_OPS - (3 * s.STDDEV_OPS)                                AS SD_LOWER_3,
    s.MEAN_OPS + (3 * s.STDDEV_OPS)                                AS SD_UPPER_3,
    s.Q1_OPS,
    s.Q3_OPS,
    (s.Q3_OPS - s.Q1_OPS)                                          AS IQR,
    s.Q1_OPS - (1.5 * (s.Q3_OPS - s.Q1_OPS))                       AS IQR_LOWER_FENCE,
    s.Q3_OPS + (1.5 * (s.Q3_OPS - s.Q1_OPS))                       AS IQR_UPPER_FENCE,
    CASE
        WHEN s.SAMPLE_DAYS < 5 THEN 'LOW CONFIDENCE - FEWER THAN 5 LOAD DAYS IN HISTORY'
        WHEN t.TODAY_OPS <= s.MEAN_OPS - (3 * s.STDDEV_OPS)
          OR t.TODAY_OPS >= s.MEAN_OPS + (3 * s.STDDEV_OPS)        THEN 'SD ALERT - 3 STDDEV'
        WHEN t.TODAY_OPS <= s.MEAN_OPS - (2 * s.STDDEV_OPS)
          OR t.TODAY_OPS >= s.MEAN_OPS + (2 * s.STDDEV_OPS)        THEN 'SD ALERT - 2 STDDEV'
        ELSE 'OK - WITHIN SD RANGE'
    END                                                             AS SD_FLAG,
    CASE
        WHEN s.SAMPLE_DAYS < 5 THEN 'LOW CONFIDENCE - FEWER THAN 5 LOAD DAYS IN HISTORY'
        WHEN t.TODAY_OPS < s.Q1_OPS - (1.5 * (s.Q3_OPS - s.Q1_OPS))
          OR t.TODAY_OPS > s.Q3_OPS + (1.5 * (s.Q3_OPS - s.Q1_OPS)) THEN 'IQR OUTLIER'
        ELSE 'OK - WITHIN IQR RANGE'
    END                                                             AS IQR_FLAG
FROM today t
CROSS JOIN stats s;

-- What to look for:
-- SD_FLAG and IQR_FLAG are reported SEPARATELY and intentionally do not
-- always agree — that is the "clearly differentiate between SD-based
-- deviations and IQR-based outlier detection" requirement from Siddesh's
-- doc. A table can fail one test and pass the other; both are valid
-- signals, investigate either flag.
-- SAMPLE_DAYS is the number of distinct days with activity in the last
-- 30 days, NOT 30 minus weekends or any other adjustment. A WEEKLY or
-- MONTHLY table will legitimately have a low SAMPLE_DAYS — that is
-- expected, not a data problem. Below 5 days, mean/stddev/percentiles
-- are not statistically meaningful, so both flags report LOW CONFIDENCE
-- instead of a real verdict. Do not treat LOW CONFIDENCE as a pass or
-- a fail — it means "this query can't tell you, check manually" (e.g.
-- with Q2's load-history drill-down instead).
-- IMPORTANT: this history is built from days that HAD activity, not a
-- zero-filled calendar. For a WEEKLY table, "mean daily volume" means
-- "mean volume on the day(s) it loads," not "mean volume per calendar
-- day including silent days." Keep that in mind when reading MEAN_OPS.


-- ──────────────────────────────────────────────────────────────────────
-- Q8 — CT Table Load Distribution Ranking (Siddesh Step 6)
-- ──────────────────────────────────────────────────────────────────────
-- Gap being closed: nothing in the existing doc ranks ALL CT tables
-- (across all schemas) by today's volume to surface the highest and
-- lowest activity tables in one shot. Q7 above is single-table; this is
-- the multi-table sweep Siddesh's Step 6 describes. Uses the same
-- LISTAGG dynamic-SQL pattern as the existing Q1 so it covers every
-- schema without hardcoding table names.

EXECUTE IMMEDIATE
$$
DECLARE
    sql_stmt STRING;
    rs RESULTSET;
BEGIN

    -- NOTE: filter to today's rows FIRST (in the WHERE clause), then
    -- aggregate. Filtering inside CASE/COUNT expressions instead of WHERE
    -- forces a full scan of every row in every CT table (some have
    -- millions of rows) on every run. Filtering in WHERE lets Snowflake
    -- prune using HEADER__TIMESTAMP before aggregating.
    SELECT LISTAGG(
        'SELECT ''' || TABLE_SCHEMA || ''' AS SCHEMA_NAME, ''' ||
        TABLE_NAME || ''' AS TABLE_NAME, ' ||
        'SUM(CASE WHEN HEADER__OPERATION = ''INSERT'' THEN 1 ELSE 0 END) AS INSERTS_TODAY, ' ||
        'SUM(CASE WHEN HEADER__OPERATION = ''UPDATE'' THEN 1 ELSE 0 END) AS UPDATES_TODAY, ' ||
        'SUM(CASE WHEN HEADER__OPERATION = ''DELETE'' THEN 1 ELSE 0 END) AS DELETES_TODAY, ' ||
        'COUNT(*) AS TOTAL_TODAY ' ||
        'FROM PROD_LANDING.' || TABLE_SCHEMA || '.' || TABLE_NAME || ' ' ||
        'WHERE TO_DATE(HEADER__TIMESTAMP) = CURRENT_DATE()',
        ' UNION ALL '
    )
    INTO :sql_stmt
    FROM PROD_LANDING.INFORMATION_SCHEMA.TABLES
    WHERE TABLE_TYPE = 'BASE TABLE'
      AND TABLE_SCHEMA IN ('ING','FRATDB','RPLUS','FRAT_ING','DI','DMC','L70','DCLM','SEI','SF','SAP','LTC')
      AND TABLE_NAME LIKE '%\_CT' ESCAPE '\';

    sql_stmt := 'SELECT * FROM (' || :sql_stmt || ') ORDER BY TOTAL_TODAY ASC';

    rs := (EXECUTE IMMEDIATE :sql_stmt);
    RETURN TABLE(rs);

END;
$$;

-- What to look for:
-- Results are sorted ascending by TOTAL_TODAY, so the lowest-activity
-- tables appear first and highest-activity last (scroll to bottom).
-- Per Siddesh's doc: "both [very low and very high] can indicate
-- potential issues" — investigate both ends, not just zeros.
-- Tables with TOTAL_TODAY = 0 will also be caught by Q1's staleness
-- check / the new Q9 below; this query's purpose is the relative
-- ranking, not staleness detection.


-- ──────────────────────────────────────────────────────────────────────
-- Q9 — Missing-Load Detection: Expected Gaps vs Unexpected Misses
--       (Siddesh Step 8)
-- ──────────────────────────────────────────────────────────────────────
-- Gap being closed: existing Q1 has toggles for "not loaded today / 7
-- days / 30 days" but does NOT classify a miss as expected (e.g. table
-- is WEEKLY and today isn't its day) vs unexpected (table is DAILY and
-- should have loaded). This reuses Q1's own LOAD_PATTERN logic as the
-- "expected schedule" reference, per the doc's instruction to "use the
-- validation query to check load frequency and schedule."

EXECUTE IMMEDIATE
$$
DECLARE
    sql_stmt STRING;
    rs RESULTSET;
BEGIN

    -- NOTE: bounded to a 90-day window in WHERE (pattern classification
    -- needs up to 90 days for MONTHLY detection — same window Q1 already
    -- uses). Without this bound, every CT table's FULL history gets
    -- scanned on every run, which is expensive at scale (some CT tables
    -- have millions of rows going back further than 90 days).
    SELECT LISTAGG(
        'SELECT ''' || TABLE_SCHEMA || ''' AS SCHEMA_NAME, ''' ||
        TABLE_NAME || ''' AS TABLE_NAME, ' ||
        'MAX(TO_DATE(HEADER__TIMESTAMP)) AS LAST_LOAD_DATE, ' ||
        'DATEDIFF(day, MAX(TO_DATE(HEADER__TIMESTAMP)), CURRENT_DATE()) AS DAYS_STALE, ' ||
        'CASE ' ||
        '  WHEN COUNT(DISTINCT CASE WHEN TO_DATE(HEADER__TIMESTAMP) >= CURRENT_DATE() - 30 THEN TO_DATE(HEADER__TIMESTAMP) END) >= 25 THEN ''DAILY'' ' ||
        '  WHEN COUNT(DISTINCT CASE WHEN TO_DATE(HEADER__TIMESTAMP) >= CURRENT_DATE() - 30 THEN TO_DATE(HEADER__TIMESTAMP) END) BETWEEN 15 AND 24 THEN ''NEAR_DAILY'' ' ||
        '  WHEN COUNT(DISTINCT CASE WHEN TO_DATE(HEADER__TIMESTAMP) >= CURRENT_DATE() - 60 THEN TO_DATE(HEADER__TIMESTAMP) END) BETWEEN 6 AND 14 THEN ''WEEKLY'' ' ||
        '  WHEN COUNT(DISTINCT CASE WHEN TO_DATE(HEADER__TIMESTAMP) >= CURRENT_DATE() - 90 THEN TO_DATE(HEADER__TIMESTAMP) END) BETWEEN 2 AND 5 THEN ''MONTHLY'' ' ||
        '  ELSE ''IRREGULAR'' ' ||
        'END AS LOAD_PATTERN, ' ||
        'MAX(CASE WHEN TO_DATE(HEADER__TIMESTAMP) = CURRENT_DATE() THEN 1 ELSE 0 END) AS LOADED_TODAY ' ||
        'FROM PROD_LANDING.' || TABLE_SCHEMA || '.' || TABLE_NAME || ' ' ||
        'WHERE TO_DATE(HEADER__TIMESTAMP) >= CURRENT_DATE() - 90',
        ' UNION ALL '
    )
    INTO :sql_stmt
    FROM PROD_LANDING.INFORMATION_SCHEMA.TABLES
    WHERE TABLE_TYPE = 'BASE TABLE'
      AND TABLE_SCHEMA IN ('ING','FRATDB','RPLUS','FRAT_ING','DI','DMC','L70','DCLM','SEI','SF','SAP','LTC')
      AND TABLE_NAME LIKE '%\_CT' ESCAPE '\';

    sql_stmt :=
        'SELECT *, CASE ' ||
        '  WHEN LOADED_TODAY = 1 THEN ''LOADED TODAY'' ' ||
        '  WHEN DAYS_STALE IS NULL THEN ''UNEXPECTED MISS - NO ACTIVITY IN 90+ DAYS'' ' ||
        '  WHEN LOAD_PATTERN = ''DAILY''      THEN ''UNEXPECTED MISS - DAILY TABLE'' ' ||
        '  WHEN LOAD_PATTERN = ''NEAR_DAILY'' AND DAYS_STALE > 2  THEN ''UNEXPECTED MISS - NEAR-DAILY TABLE'' ' ||
        '  WHEN LOAD_PATTERN = ''NEAR_DAILY'' THEN ''EXPECTED GAP - NEAR-DAILY TABLE'' ' ||
        '  WHEN LOAD_PATTERN = ''WEEKLY''  AND DAYS_STALE > 10 THEN ''UNEXPECTED MISS - WEEKLY TABLE'' ' ||
        '  WHEN LOAD_PATTERN = ''WEEKLY''  THEN ''EXPECTED GAP - WEEKLY TABLE'' ' ||
        '  WHEN LOAD_PATTERN = ''MONTHLY'' AND DAYS_STALE > 40 THEN ''UNEXPECTED MISS - MONTHLY TABLE'' ' ||
        '  WHEN LOAD_PATTERN = ''MONTHLY'' THEN ''EXPECTED GAP - MONTHLY TABLE'' ' ||
        '  WHEN LOAD_PATTERN = ''IRREGULAR'' THEN ''CHECK MANUALLY - IRREGULAR PATTERN'' ' ||
        '  ELSE ''REVIEW'' ' ||
        'END AS MISS_CLASSIFICATION ' ||
        'FROM (' || :sql_stmt || ') ' ||
        'WHERE LOADED_TODAY = 0 ' ||           -- only show tables NOT loaded today
        'ORDER BY DAYS_STALE DESC NULLS FIRST';

    rs := (EXECUTE IMMEDIATE :sql_stmt);
    RETURN TABLE(rs);

END;
$$;

-- What to look for:
-- Only tables NOT loaded today are returned. MISS_CLASSIFICATION tells
-- you whether the gap is expected (matches the table's own historical
-- pattern from Q1's logic) or unexpected (table is overdue against its
-- own pattern). "UNEXPECTED MISS" rows are the priority — they mean a
-- table that should have loaded today, by its own history, did not.
-- "CHECK MANUALLY - IRREGULAR PATTERN" tables have no clear cadence;
-- per the existing doc, confirm with Mary Beth/Josh whether expected.
-- This reuses Q1's exact pattern thresholds so classification stays
-- consistent with the staleness query already in the doc.
-- CAVEAT: the underlying scan is bounded to the last 90 days for cost
-- reasons (same bound used elsewhere in this doc). A table that hasn't
-- loaded in 91+ days will show LAST_LOAD_DATE and DAYS_STALE as NULL
-- here, NOT its true last-load date — NULL in this query means "no
-- activity in the last 90 days," not "never loaded." For tables that
-- stale, Q1's full-history staleness check is the authoritative source;
-- this query is only meant to catch TODAY's misses.


-- ══════════════════════════════════════════════════════════════════════
-- ON FMC "RECORDS PROCESSED" (Siddesh Step 9, second half)
-- ══════════════════════════════════════════════════════════════════════
-- NOT BUILT. Confirmed via INFORMATION_SCHEMA.COLUMNS that neither
-- FMC_LOAD_BATCH_HISTORY nor FMC_OBJECT_LOADING_HISTORY contains a
-- record-count column. Columns present: DAG_NAME, BKCC, DV_LOAD_BATCH_ID,
-- DV_LOAD_TIMESTAMP, FMC_BEGIN_LW_TIMESTAMP, FMC_END_LW_TIMESTAMP,
-- LOAD_START_DATE, LOAD_END_DATE, SUCCESS_FLAG (+ DV_SOURCE_OBJECT_NAME
-- on the object-level table only).
--
-- The DAG RUNTIME half of Step 9 (2-3 stddev on elapsed time) is already
-- built and in production use — see Datamod Monitoring SOP r1.docx,
-- "Step 3: Check for Failed, Long-Running, or Hung Jobs" section. That
-- query is not duplicated here to avoid two diverging versions of the
-- same logic.
--
-- If records-processed monitoring is still required, the only path is
-- counting rows in FMC_OBJECT_LOADING_HISTORY per DV_LOAD_BATCH_ID as a
-- proxy (count of OBJECTS loaded in a batch, not count of ROWS/RECORDS
-- within each object). That is a materially different metric than
-- "records processed" and should not be presented as equivalent without
-- confirming with Siddesh first.
