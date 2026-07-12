
/* ============================================================================
   T8LPUA — BACKTRACKING THE CRNT_REC_IND DUPLICATE-FLAG ISSUE
   ----------------------------------------------------------------------------
   FINDING (confirmed): 19 business keys in KCDWHPRC.T8LPUA have MORE THAN
   ONE row simultaneously flagged CRNT_REC_IND = 'Y' for the same key:
       SOURCE, CO_ID, POL_ID, CVG_NUM, DIV_EFF_DT, CVG_DIV_OPT_CD, L7FD_PECD
   These rows come from multiple different RPT_PRCES_STATS_ID values and
   span both historical and recent data. Confirmed NOT present last quarter.

   WHAT THIS PATTERN MEANS:
     T8LPUA looks like a Type-2 history table - each time a policy's PUA
     data changes, the OLD row should be expired (CRNT_REC_IND set to 'N')
     and a NEW row inserted with CRNT_REC_IND = 'Y'. At any moment, exactly
     ONE row per business key should show 'Y'.

     Finding more than one 'Y' per key means the "expire the old row"
     step is failing for these 19 keys - either the load process has a
     bug, or something (a rerun, a backfill, a partial reprocess) touched
     these specific rows without properly closing out the previous
     version.

   GOAL OF THIS SCRIPT:
     Build a precise timeline for each of the 19 keys - which rows exist,
     which RPT_PRCES_STATS_ID loaded each one, and when - so we can see
     WHEN this started and whether it traces to a specific batch run,
     a rerun, or a backfill event.

   NOTE ON COLUMN NAMES:
     RPT_PRCES_STATS_ID is confirmed to exist on T8LPUA (per your
     finding). This script also assumes there is SOME kind of load/create
     timestamp column on the table (common patterns: LOAD_TS, CREATE_TS,
     LOAD_DT, INSRT_TS). Adjust the marked (!) lines to match the actual
     column name once confirmed - the logic doesn't change, just the
     column reference.
============================================================================ */


/* ============================================================================
   STEP A — Get full row-level detail for the 19 flagged keys
   ----------------------------------------------------------------------------
   Not just counts this time - every column for every row involved, so we
   can see the actual data differences between the duplicate 'Y' rows.
============================================================================ */

WITH FLAGGED_KEYS AS (
    SELECT SOURCE, CO_ID, POL_ID, CVG_NUM, DIV_EFF_DT, CVG_DIV_OPT_CD, L7FD_PECD
    FROM KCDWHPRC.T8LPUA
    WHERE CRNT_REC_IND = 'Y'
    GROUP BY SOURCE, CO_ID, POL_ID, CVG_NUM, DIV_EFF_DT, CVG_DIV_OPT_CD, L7FD_PECD
    HAVING COUNT(*) > 1
)
SELECT
    T.SOURCE, T.CO_ID, T.POL_ID, T.CVG_NUM, T.DIV_EFF_DT,
    T.CVG_DIV_OPT_CD, T.L7FD_PECD,
    T.CRNT_REC_IND,
    T.RPT_PRCES_STATS_ID,
    T.LOAD_TS,                                  -- (!) adjust column name if different
    T.PUA_LTD_FACE_AMT,
    T.INSID_PUA_LTD_AMT,
    T.PUA_CLR_FACE_AMT,
    T.CVG_DOD_ACUM_AMT
FROM KCDWHPRC.T8LPUA T
INNER JOIN FLAGGED_KEYS FK
    ON T.SOURCE = FK.SOURCE
    AND T.CO_ID = FK.CO_ID
    AND T.POL_ID = FK.POL_ID
    AND T.CVG_NUM = FK.CVG_NUM
    AND T.DIV_EFF_DT = FK.DIV_EFF_DT
    AND T.CVG_DIV_OPT_CD = FK.CVG_DIV_OPT_CD
    AND T.L7FD_PECD = FK.L7FD_PECD
WHERE T.CRNT_REC_IND = 'Y'
ORDER BY T.CO_ID, T.POL_ID, T.CVG_NUM, T.LOAD_TS;   -- (!) adjust column name


/* ============================================================================
   STEP B — Which RPT_PRCES_STATS_ID values are involved, and when
   ----------------------------------------------------------------------------
   This tells us whether the 19 keys cluster around ONE problematic run
   (pointing to a single bad job execution) or are scattered across MANY
   different runs (pointing to a systemic logic bug, not a one-time event).
============================================================================ */

WITH FLAGGED_KEYS AS (
    SELECT SOURCE, CO_ID, POL_ID, CVG_NUM, DIV_EFF_DT, CVG_DIV_OPT_CD, L7FD_PECD
    FROM KCDWHPRC.T8LPUA
    WHERE CRNT_REC_IND = 'Y'
    GROUP BY SOURCE, CO_ID, POL_ID, CVG_NUM, DIV_EFF_DT, CVG_DIV_OPT_CD, L7FD_PECD
    HAVING COUNT(*) > 1
)
SELECT
    T.RPT_PRCES_STATS_ID,
    MIN(T.LOAD_TS) AS EARLIEST_LOAD_IN_THIS_RUN,     -- (!) adjust column name
    MAX(T.LOAD_TS) AS LATEST_LOAD_IN_THIS_RUN,       -- (!) adjust column name
    COUNT(*) AS ROWS_FROM_THIS_RUN,
    COUNT(DISTINCT T.CO_ID || '-' || T.POL_ID) AS DISTINCT_POLICIES_AFFECTED
FROM KCDWHPRC.T8LPUA T
INNER JOIN FLAGGED_KEYS FK
    ON T.SOURCE = FK.SOURCE
    AND T.CO_ID = FK.CO_ID
    AND T.POL_ID = FK.POL_ID
    AND T.CVG_NUM = FK.CVG_NUM
    AND T.DIV_EFF_DT = FK.DIV_EFF_DT
    AND T.CVG_DIV_OPT_CD = FK.CVG_DIV_OPT_CD
    AND T.L7FD_PECD = FK.L7FD_PECD
WHERE T.CRNT_REC_IND = 'Y'
GROUP BY T.RPT_PRCES_STATS_ID
ORDER BY EARLIEST_LOAD_IN_THIS_RUN;

/* HOW TO READ THIS:
   - If all 19 keys trace back to a SMALL number of RPT_PRCES_STATS_ID
     values (ideally just 1-2), that points to a specific bad run -
     something unusual happened during that particular execution.
   - If they're scattered across many different STATS_IDs over time,
     that points to a persistent logic bug in the load process itself -
     it's been failing to expire old rows on a recurring basis, and
     these 19 are just the ones that happened to also get reported into
     POL_VALU this quarter.
*/


/* ============================================================================
   STEP C — Full history for ONE example key (deep dive)
   ----------------------------------------------------------------------------
   Pick one of the 19 keys from Step A and pull EVERY row ever loaded for
   it (not just the CRNT_REC_IND='Y' ones) - this shows the complete
   Type-2 history chain and exactly where it breaks.
============================================================================ */

SELECT
    SOURCE, CO_ID, POL_ID, CVG_NUM, DIV_EFF_DT, CVG_DIV_OPT_CD, L7FD_PECD,
    CRNT_REC_IND,
    RPT_PRCES_STATS_ID,
    LOAD_TS,                                    -- (!) adjust column name
    PUA_LTD_FACE_AMT, INSID_PUA_LTD_AMT, PUA_CLR_FACE_AMT
FROM KCDWHPRC.T8LPUA
WHERE SOURCE = 'PASTE_SOURCE_HERE'
  AND CO_ID = 'PASTE_CO_ID_HERE'
  AND POL_ID = 'PASTE_POL_ID_HERE'
  AND CVG_NUM = 'PASTE_CVG_NUM_HERE'
  AND DIV_EFF_DT = 'PASTE_DIV_EFF_DT_HERE'
  AND CVG_DIV_OPT_CD = 'PASTE_CVG_DIV_OPT_CD_HERE'
  AND L7FD_PECD = 'PASTE_L7FD_PECD_HERE'
ORDER BY LOAD_TS;                               -- (!) adjust column name

/* WHAT TO LOOK FOR:
   A HEALTHY history chain looks like:
     Row 1: CRNT_REC_IND='N', loaded on Date1, STATS_ID=100
     Row 2: CRNT_REC_IND='N', loaded on Date2, STATS_ID=150
     Row 3: CRNT_REC_IND='Y', loaded on Date3, STATS_ID=200   <- only the latest is 'Y'

   A BROKEN chain (what we're hunting) looks like:
     Row 1: CRNT_REC_IND='Y', loaded on Date1, STATS_ID=100   <- should have been expired
     Row 2: CRNT_REC_IND='N', loaded on Date2, STATS_ID=150
     Row 3: CRNT_REC_IND='Y', loaded on Date3, STATS_ID=200   <- correctly current

   If Row 1 is still 'Y' when it should have been flipped to 'N' at the
   time Row 3 was loaded, that's your smoking gun - the load process
   that ran at STATS_ID=200 (or whichever run inserted the newest 'Y')
   failed to expire the earlier one.
*/


/* ============================================================================
   STEP D — Scope check: is 19 the whole problem, or just what reached POL_VALU?
   ----------------------------------------------------------------------------
   POL_VALU only sees rows that pass the report's own filters (DIV_EFF_DT
   window, CRNT_REC_IND='Y', etc.) There may be MORE broken keys in
   T8LPUA that never surfaced in POL_VALU because they fall outside this
   quarter's report window. Worth knowing the true blast radius.
============================================================================ */

SELECT COUNT(*) AS TOTAL_KEYS_WITH_MULTIPLE_CURRENT_FLAGS
FROM (
    SELECT SOURCE, CO_ID, POL_ID, CVG_NUM, DIV_EFF_DT, CVG_DIV_OPT_CD, L7FD_PECD
    FROM KCDWHPRC.T8LPUA
    WHERE CRNT_REC_IND = 'Y'
    GROUP BY SOURCE, CO_ID, POL_ID, CVG_NUM, DIV_EFF_DT, CVG_DIV_OPT_CD, L7FD_PECD
    HAVING COUNT(*) > 1
);

-- If this number is bigger than 19, the report window is only showing
-- you part of the problem - worth flagging that distinction when you
-- report this back to Suresh/Siddesh.


/* ============================================================================
   STEP E — Confirm "not present last quarter" empirically
   ----------------------------------------------------------------------------
   Rather than relying on memory, use RPT_PRCES_STATS_ID / LOAD_TS to see
   whether the earliest offending row for any of the 19 keys falls inside
   this quarter's window specifically.
============================================================================ */

WITH FLAGGED_KEYS AS (
    SELECT SOURCE, CO_ID, POL_ID, CVG_NUM, DIV_EFF_DT, CVG_DIV_OPT_CD, L7FD_PECD
    FROM KCDWHPRC.T8LPUA
    WHERE CRNT_REC_IND = 'Y'
    GROUP BY SOURCE, CO_ID, POL_ID, CVG_NUM, DIV_EFF_DT, CVG_DIV_OPT_CD, L7FD_PECD
    HAVING COUNT(*) > 1
)
SELECT
    MIN(T.LOAD_TS) AS EARLIEST_OFFENDING_LOAD,        -- (!) adjust column name
    MAX(T.LOAD_TS) AS LATEST_OFFENDING_LOAD           -- (!) adjust column name
FROM KCDWHPRC.T8LPUA T
INNER JOIN FLAGGED_KEYS FK
    ON T.SOURCE = FK.SOURCE AND T.CO_ID = FK.CO_ID AND T.POL_ID = FK.POL_ID
    AND T.CVG_NUM = FK.CVG_NUM AND T.DIV_EFF_DT = FK.DIV_EFF_DT
    AND T.CVG_DIV_OPT_CD = FK.CVG_DIV_OPT_CD AND T.L7FD_PECD = FK.L7FD_PECD
WHERE T.CRNT_REC_IND = 'Y';

-- If EARLIEST_OFFENDING_LOAD falls within this quarter (Q2), that
-- empirically confirms this is a new regression, not something that
-- was always there and only just got noticed.


/* ============================================================================
   NEXT STEPS ONCE THIS COMES BACK
   ----------------------------------------------------------------------------
   1. If Step B shows the 19 keys cluster around 1-2 RPT_PRCES_STATS_ID
      values -> ask the DBA/job team what happened during those specific
      runs (rerun? backfill? manual fix attempt? job failure and retry?).
      This is the same style of question Suresh walked through for the
      DCLM incident - "was the correct file/data used for this run".

   2. If Step B shows many scattered STATS_IDs -> this points to the
      load job's "expire previous record" UPDATE statement itself having
      a bug (wrong WHERE clause, missing a key column, race condition
      between concurrent loads) - worth finding the actual body job that
      writes to T8LPUA and reviewing that specific UPDATE logic.

   3. Worth checking against the November DR/reprocessing event
      mentioned in earlier meeting notes (a month's load was "completely
      missed" during a disaster-recovery restore and had to be rerun) -
      partial reprocessing is a very common cause of exactly this
      orphaned-flag pattern. If any of the 19 keys' DIV_EFF_DT or
      RPT_PRCES_STATS_ID lines up with a known reprocessing window,
      that's a strong lead.

   4. Once you know WHICH run/timeframe caused this, that's what lets you
      go find the specific body job (same approach as before - search
      the Data Services Management Console by target table T8LPUA) and
      check its logic and run history for that specific date.
============================================================================ */
