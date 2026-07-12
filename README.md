
Two things worth checking next to narrow this down
1. Does failure correlate with "busy" runs? Look at Step B again: the runs with more rows/more policies (4346202 → 6 rows/6 policies, 4357177 → 5 rows/3 policies) are exactly the kind of larger, multi-policy batch runs. If failures cluster specifically on the "big" runs and the single-row runs mostly succeed, that points to a bug in the batch/set-based UPDATE logic (e.g., matching on the wrong subset of key columns when processing many records at once) rather than a pure timing race.
sql-- Does failure rate correlate with how many rows a run touched?
-- (uses your existing Step B output logic, just re-grouped for this lens)
SELECT
    CASE WHEN ROWS_FROM_THIS_RUN = 1 THEN 'SINGLE_ROW_RUN' ELSE 'MULTI_ROW_RUN' END AS RUN_TYPE,
    COUNT(*) AS NUM_RUNS,
    SUM(ROWS_FROM_THIS_RUN) AS TOTAL_ORPHANED_ROWS
FROM (
    -- paste your Step B result set here, or re-run it as a CTE
    SELECT RPT_PRCES_STATS_ID, COUNT(*) AS ROWS_FROM_THIS_RUN
    FROM KCDWHPRC.T8LPUA T
    -- (your existing FLAGGED_KEYS join logic)
    GROUP BY RPT_PRCES_STATS_ID
) X
GROUP BY CASE WHEN ROWS_FROM_THIS_RUN = 1 THEN 'SINGLE_ROW_RUN' ELSE 'MULTI_ROW_RUN' END;
2. Test explanation #1 directly — would last quarter's exact report window have caught these same orphaned rows?
sql-- Substitute last quarter's actual RPT_DT below.
-- If this returns the SAME or a SIMILAR set of keys, the defect was
-- there last quarter too — it just wasn't looked at with this check.
-- If it returns ZERO, last quarter's window genuinely excluded these
-- specific orphaned rows (supporting "newly visible, not newly broken").
SELECT SOURCE, CO_ID, POL_ID, CVG_NUM, DIV_EFF_DT, CVG_DIV_OPT_CD, L7FD_PECD, COUNT(*)
FROM KCDWHPRC.T8LPUA
WHERE CRNT_REC_IND = 'Y'
  AND DIV_EFF_DT >= DATE('PASTE_LAST_QUARTER_RPT_DT') - 1 YEAR   -- (!) last quarter's window
GROUP BY SOURCE, CO_ID, POL_ID, CVG_NUM, DIV_EFF_DT, CVG_DIV_OPT_CD, L7FD_PECD
HAVING COUNT(*) > 1;
