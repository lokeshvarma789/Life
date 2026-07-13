-- See the actual repeat pattern - how many times does each amount repeat?
SELECT CO, POLICY, ENTRDT, ACCT_TRXN_CD, SUNDAMT, COUNT(*) AS TIMES_REPEATED
FROM KCDWHPRC.T8D69
WHERE CO = '01' AND POLICY = '0293027'
  AND ENTRDT = '2026-06-11' AND ACCT_TRXN_CD = '012'
GROUP BY CO, POLICY, ENTRDT, ACCT_TRXN_CD, SUNDAMT
ORDER BY TIMES_REPEATED DESC;
T8VDV, T9DVP, T8LPOLL — this is the real finding
All three carry CRNT_REC_IND/EFF_DT/END_DT — the exact same Type-2 pattern as T8LPUA. This is worth escalating regardless of what the next queries show, because it means the same expire-and-replace mechanism exists in multiple CDC jobs, not just T8LPUA_CDC_LOAD.
But first, a correction: my original duplicate queries for T9DVP and T8LPOLL both included EFF_DT in the GROUP BY. That's the exact mistake — EFF_DT is the version-control column that differs between the old and new version of the same record. Grouping by it means two rows that should collide (same business key, both wrongly marked current) get sorted into separate groups and never get compared against each other. Those two earlier checks were structurally blind to this bug. T8VDV's original query didn't have this problem — it never included EFF_DT.
Corrected queries — run these three:
sql-- T8VDV — original query was already correct, re-run as-is
SELECT CO_ID, POL_ID, CVG_NUM, CVG_DIV_DT, COUNT(*) AS DUP_COUNT
FROM KCDWHPRC.T8VDV
WHERE CRNT_REC_IND = 'Y'
GROUP BY CO_ID, POL_ID, CVG_NUM, CVG_DIV_DT
HAVING COUNT(*) > 1;
sql-- T9DVP — CORRECTED: removed EFF_DT from the grouping
SELECT CO_ID, POL_ID, REC_TYP_CD, COUNT(*) AS DUP_COUNT
FROM KCDWHPRC.T9DVP
WHERE CRNT_REC_IND = 'Y' AND DERIV_VALU_FREQ_CD = 'C'
GROUP BY CO_ID, POL_ID, REC_TYP_CD
HAVING COUNT(*) > 1;
sql-- T8LPOLL — CORRECTED: removed EFF_DT from the grouping
SELECT CO_ID, POL_ID, COUNT(*) AS DUP_COUNT
FROM KCDWHPRC.T8LPOLL
WHERE CRNT_REC_IND = 'Y'
  AND CO_ID IN ('01', '50') AND POL_ID > '0'
GROUP BY CO_ID, POL_ID
HAVING COUNT(*) > 1;
