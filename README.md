
/* ============================================================================
   WALKTHROUGH — SHOW THIS LIVE TO SIDDESH / SURESH
   ----------------------------------------------------------------------------
   Five steps. Run each one, look at the result, then read the comment
   right above it out loud before moving to the next.
============================================================================ */


/* ============================================================================
   STEP 1 — "Here's the actual problem. This is the query that finds it."
   ----------------------------------------------------------------------------
   T8LPUA keeps a running history of every policy's dividend records.
   Each policy should have EXACTLY ONE row marked "current" (Y) at any
   moment - every older version should be marked "no longer current" (N).

   This query looks for any policy where MORE THAN ONE row is marked
   current at the same time. That should never happen. If it returns
   any rows, those are the ones causing the duplicates.
============================================================================ */

SELECT SOURCE, CO_ID, POL_ID, CVG_NUM, DIV_EFF_DT, CVG_DIV_OPT_CD, L7FD_PECD,
       COUNT(*) AS HOW_MANY_ROWS_MARKED_CURRENT
FROM KCDWHPRC.T8LPUA
WHERE CRNT_REC_IND = 'Y'
GROUP BY SOURCE, CO_ID, POL_ID, CVG_NUM, DIV_EFF_DT, CVG_DIV_OPT_CD, L7FD_PECD
HAVING COUNT(*) > 1
ORDER BY POL_ID;

-- RESULT: 19 policies come back here. Every one of them has 2 rows
-- both marked "current" when there should only be 1.
-- SAY: "These 19 policies are the actual source. The report isn't
-- creating these duplicates - it's just showing us what's already
-- sitting in this table."


/* ============================================================================
   STEP 2 — "Let's look at one of them up close."
   ----------------------------------------------------------------------------
   Pick one policy from Step 1's results and pull both of its "current"
   rows side by side so we can see them with our own eyes.
============================================================================ */

SELECT L70PUA_FCT_ID, CO_ID, POL_ID, CVG_NUM, DIV_EFF_DT, L7FD_PECD,
       CRNT_REC_IND, RPT_PRCES_STATS_ID, CREAT_TS
FROM KCDWHPRC.T8LPUA
WHERE CO_ID = '01' AND POL_ID = '01120182'
  AND CVG_NUM = '01' AND L7FD_PECD = '0'
  AND CRNT_REC_IND = 'Y'
ORDER BY CREAT_TS;

-- RESULT: two rows come back, same policy, same everything except
-- they were loaded by two different runs, months apart.
-- SAY: "This is one policy. Both of these rows say 'I am the current
-- version.' They can't both be right."


/* ============================================================================
   STEP 3 — "Now the important question: did this happen in ONE run,
   or across TWO separate runs?"
   ----------------------------------------------------------------------------
   This matters because it tells us what KIND of problem we're dealing
   with. If it's one run, it means two records collided with each other
   in a single load. If it's two separate runs, it means a LATER run
   failed to notice an EARLIER run's record was already there.

   RPT_PRCES_STATS_ID is basically "which load/run created this row."
   We're counting how many different runs are involved for each of our
   19 problem policies.
============================================================================ */

WITH FLAGGED_KEYS AS (
    SELECT SOURCE, CO_ID, POL_ID, CVG_NUM, DIV_EFF_DT, CVG_DIV_OPT_CD, L7FD_PECD
    FROM KCDWHPRC.T8LPUA
    WHERE CRNT_REC_IND = 'Y'
    GROUP BY SOURCE, CO_ID, POL_ID, CVG_NUM, DIV_EFF_DT, CVG_DIV_OPT_CD, L7FD_PECD
    HAVING COUNT(*) > 1
)
SELECT
    T.CO_ID, T.POL_ID, T.DIV_EFF_DT,
    COUNT(DISTINCT T.RPT_PRCES_STATS_ID) AS HOW_MANY_DIFFERENT_RUNS,
    MIN(T.RPT_PRCES_STATS_ID) AS EARLIER_RUN,
    MAX(T.RPT_PRCES_STATS_ID) AS LATER_RUN
FROM KCDWHPRC.T8LPUA T
INNER JOIN FLAGGED_KEYS FK
    ON T.SOURCE = FK.SOURCE AND T.CO_ID = FK.CO_ID AND T.POL_ID = FK.POL_ID
    AND T.CVG_NUM = FK.CVG_NUM AND T.DIV_EFF_DT = FK.DIV_EFF_DT
    AND T.CVG_DIV_OPT_CD = FK.CVG_DIV_OPT_CD AND T.L7FD_PECD = FK.L7FD_PECD
WHERE T.CRNT_REC_IND = 'Y'
GROUP BY T.CO_ID, T.POL_ID, T.DIV_EFF_DT
ORDER BY HOW_MANY_DIFFERENT_RUNS DESC;

-- RESULT: every single one of the 19 shows "2 different runs" - never 1.
-- SAY: "This tells us it's NOT two records colliding in the same load.
-- It's a LATER run that failed to see a record from an EARLIER run and
-- properly retire it. That's an important distinction - it means we're
-- not looking at a design flaw, we're looking at something that went
-- wrong during specific individual runs."


/* ============================================================================
   STEP 4 — "The most convincing part: watch this SAME policy work
   correctly, over and over, for years - and then fail exactly once."
   ----------------------------------------------------------------------------
   This pulls the ENTIRE history for one policy, going back as far as
   the data goes. Every time this ran successfully in the past, you'll
   see the old row correctly flip to "N" right when the new row comes
   in as "Y." Watch for the one place where that DOESN'T happen.
============================================================================ */

SELECT L70PUA_FCT_ID, DIV_EFF_DT, CRNT_REC_IND, RPT_PRCES_STATS_ID, CREAT_TS
FROM KCDWHPRC.T8LPUA
WHERE CO_ID = '01' AND POL_ID = '01120182'
  AND CVG_NUM = '01' AND L7FD_PECD = '0'
ORDER BY RPT_PRCES_STATS_ID;

-- RESULT: dozens of rows spanning back to 2008. Almost every single
-- transition correctly flips the old row to N when a new one arrives.
-- There is exactly ONE place, near the bottom (most recent), where
-- this DOESN'T happen - two rows in a row both marked Y, one from
-- February 2026, one from May 2026.
-- SAY: "This is the strongest evidence we have. This exact process has
-- worked correctly dozens of times for THIS SAME POLICY, going back 17
-- years. It only failed once, in May. That tells us this isn't a
-- broken design - it's something specific that happened during that
-- one run in May. That's why we need the DBA execution logs for that
-- exact day - to see what actually happened."


/* ============================================================================
   STEP 5 — "One more thing - this isn't actually new, we just never
   checked for it before."
   ----------------------------------------------------------------------------
   Re-run Step 1's exact same check, but scoped to last quarter's
   reporting window instead of this quarter's.
============================================================================ */

SELECT SOURCE, CO_ID, POL_ID, CVG_NUM, DIV_EFF_DT, CVG_DIV_OPT_CD, L7FD_PECD,
       COUNT(*) AS HOW_MANY_ROWS_MARKED_CURRENT
FROM KCDWHPRC.T8LPUA
WHERE CRNT_REC_IND = 'Y'
  AND DIV_EFF_DT >= DATE('2026-03-31') - 1 YEAR   -- (!) swap in last quarter's actual report date
GROUP BY SOURCE, CO_ID, POL_ID, CVG_NUM, DIV_EFF_DT, CVG_DIV_OPT_CD, L7FD_PECD
HAVING COUNT(*) > 1
ORDER BY POL_ID;

-- RESULT: 15 of the same 19 policies show up here too.
-- SAY: "This confirms it was already there last quarter. It's not a
-- new problem - the normal validation just doesn't catch this kind of
-- issue, since an extra row doesn't break a record count or a date
-- check. We only found it because we went looking specifically for
-- policies with two 'current' flags."


/* ============================================================================
   WRAP-UP LINE TO CLOSE WITH
   ----------------------------------------------------------------------------
   "So to summarize: 19 policies, one source table, a later run that
   didn't properly close out an earlier record. It's worked correctly
   for years and only broke once that we can find. We need the DBA logs
   for that one specific day to close the loop on exactly why."
============================================================================ */
