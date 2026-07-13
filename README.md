
Likely FALSE POSITIVES — incomplete key, not a real bug
T9DAT — Look at how TT_RPTDT actually uses this table: it takes MAX(FINAL_DY_OF_MO_DT) and MIN(DT_VALU) across rows grouped by year/month. That only makes sense if T9DAT normally holds one row per day, not one row per month. My inferred key (CO_ID, DY_NUM_YR, CLNDR_MO_NUM) is almost certainly missing a day-level column. This is very likely the same trap as T7PIL — expected multiplicity, not a defect.
sql-- Check: does T9DAT actually have a day-level column that should be in the key?
SELECT CO_ID, DY_NUM_YR, CLNDR_MO_NUM, COUNT(*) AS ROWS,
       MIN(DT_VALU) AS EARLIEST_DAY, MAX(DT_VALU) AS LATEST_DAY,
       COUNT(DISTINCT DT_VALU) AS DISTINCT_DAYS
FROM KCDWHPRC.T9DAT
GROUP BY CO_ID, DY_NUM_YR, CLNDR_MO_NUM
HAVING COUNT(*) > 1
ORDER BY ROWS DESC
FETCH FIRST 5 ROWS ONLY;
If DISTINCT_DAYS roughly matches ROWS, that confirms it — one row per day, completely normal, false alarm.
T8D69 — This is an account transaction table. Transaction logs very commonly have multiple legitimate entries with the same code on the same day (e.g., two separate premium payments). My key (CO, POLICY, ENTRDT, ACCT_TRXN_CD) is missing whatever distinguishes individual transactions — likely a sequence number or amount.
sql-- Check: is there a transaction sequence/ID column missing from the key?
SELECT CO, POLICY, ENTRDT, ACCT_TRXN_CD, COUNT(*) AS ROWS,
       COUNT(DISTINCT SUNDAMT) AS DISTINCT_AMOUNTS
FROM KCDWHPRC.T8D69
GROUP BY CO, POLICY, ENTRDT, ACCT_TRXN_CD
HAVING COUNT(*) > 1
ORDER BY ROWS DESC
FETCH FIRST 5 ROWS ONLY;
Genuinely worth taking seriously — possible real, systemic issue
VCPUA, T8VDV, T9DVP, T8LPOLL — these are worth real attention, for a specific reason: do any of these carry a CRNT_REC_IND-style current-flag column, the same Type-2 history pattern as T8LPUA? If yes, this isn't 4 separate coincidences — it's evidence the same underlying mechanism (expire-and-replace failing intermittently) is happening across multiple sibling CDC jobs, not just T8LPUA_CDC_LOAD. That would meaningfully expand the scope of what we tell Suresh.
Recall: the job header for T8LPUA_CDC_LOAD itself mentioned T8LVDV was touched by the same 2013 change ticket — sibling jobs built from a shared template is already a known pattern here. This would be consistent with that.
sql-- Run this for each of VCPUA, T8VDV, T9DVP, T8LPOLL - does it have
-- CRNT_REC_IND, and if so, do the duplicates show the same "2 rows
-- both marked Y" pattern as T8LPUA?
SELECT COLNAME, TYPENAME
FROM SYSCAT.COLUMNS
WHERE TABSCHEMA = 'KCDWHPRC' AND TABNAME = '<table_name>'
  AND COLNAME IN ('CRNT_REC_IND', 'EFF_DT', 'END_DT');
