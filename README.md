/* ============================================================================
   ALL SOURCE TABLES — DUPLICATE CHECKS WITH CONFIRMED KEYS
   ----------------------------------------------------------------------------
   Every table used across all 10 session tables in the "new 22" /
   fr300.fex report script, organized by which session uses it.

   IMPORTANT DISTINCTION:
     - T8LPUA's key below is CONFIRMED against the real DB2 unique index
       (SYSCAT.INDEXES) - we verified this directly earlier.
     - Every other table's key below is INFERRED from how the script
       itself joins/groups on it (its own GROUP BY, its own join
       conditions). This is a reasonable starting point, but NOT yet
       confirmed against the actual database constraint the way
       T8LPUA was.

   RUN PART 1 FIRST. It pulls the REAL confirmed key for every table
   in one shot, the same way we settled the T8LPUA question. Compare
   its results against the "INFERRED KEY" noted in each Part 2 section
   before trusting any of them blindly - if a table's real index
   differs from what's inferred here, update that table's duplicate
   query to match the real key, not the inferred one.
============================================================================ */


/* ============================================================================
   PART 1 — CONFIRM THE REAL KEY FOR EVERY TABLE, ONE QUERY, ONE SHOT
   ----------------------------------------------------------------------------
   Same method used to settle the T8LPUA question. Run this once and
   keep the output next to Part 2 while reviewing each table below.
============================================================================ */

SELECT
    IX.TABNAME,
    IX.INDNAME,
    IX.UNIQUERULE,     -- 'P' = primary key, 'U' = unique index
    IXC.COLNAME,
    IXC.COLSEQ
FROM SYSCAT.INDEXES IX
JOIN SYSCAT.INDEXCOLUSE IXC
    ON IX.INDSCHEMA = IXC.INDSCHEMA AND IX.INDNAME = IXC.INDNAME
WHERE IX.TABSCHEMA = 'KCDWHPRC'
  AND IX.TABNAME IN (
      'TMAST', 'T9DAT',
      'T8LPOL', 'T7PIL', 'T8POL', 'T8PLP', 'T9PH',
      'T9LCSV', 'T8SOLIFE',
      'T8LMSG',
      'VCPUA', 'T8VDV', 'T8LPUA',
      'T9DVP', 'T9DVP_HIST',
      'TCDSA', 'TDMAV',
      'TPOLL', 'T8LPOLL', 'T8D69',
      'TCFLW'
  )
  AND IX.UNIQUERULE IN ('P', 'U')
ORDER BY IX.TABNAME, IX.INDNAME, IXC.COLSEQ;

-- NOTE: VCPUA may be a VIEW rather than a base table - if so, this
-- returns nothing for it. Views don't carry their own keys; check
-- whatever base table(s) VCPUA is built from instead if needed.


/* ============================================================================
   PART 2 — DUPLICATE CHECKS, ORGANIZED BY SESSION
============================================================================ */


/* ----------------------------------------------------------------------
   SESSION: TT_RPTDT  (the base reporting-date window - everything
   else depends on this being correct)
   ---------------------------------------------------------------------- */

-- TABLE: TMAST
-- INFERRED KEY: CO_ID, BTCH_PRCES_DT  (one batch-processing date per company)
SELECT CO_ID, BTCH_PRCES_DT, COUNT(*) AS DUP_COUNT
FROM KCDWHPRC.TMAST
GROUP BY CO_ID, BTCH_PRCES_DT
HAVING COUNT(*) > 1;

-- TABLE: T9DAT
-- INFERRED KEY: CO_ID, DY_NUM_YR, CLNDR_MO_NUM  (one calendar record per company/year/month)
SELECT CO_ID, DY_NUM_YR, CLNDR_MO_NUM, COUNT(*) AS DUP_COUNT
FROM KCDWHPRC.T9DAT
GROUP BY CO_ID, DY_NUM_YR, CLNDR_MO_NUM
HAVING COUNT(*) > 1;


/* ----------------------------------------------------------------------
   SESSION: TT_POL  (the critical one - T7PIL join lives here)
   ---------------------------------------------------------------------- */

-- TABLE: T8LPOL (LIFE70 policy master)
-- INFERRED KEY: CO_ID, POL_ID
-- FK OUT: joins to T7PIL via (CO_ID, PLAN_ID, PLAN_RTBK_CD -> RT_BK_CD)
SELECT CO_ID, POL_ID, COUNT(*) AS DUP_COUNT
FROM KCDWHPRC.T8LPOL
WHERE CRNT_REC_IND = 'Y'
GROUP BY CO_ID, POL_ID
HAVING COUNT(*) > 1;

-- TABLE: T7PIL (plan/language reference table - the confirmed fan-out source)
-- INFERRED KEY: CO_ID, PLAN_ID, RT_BK_CD, ETBL_LANG_CD
-- FK IN: referenced by T8LPOL.PLAN_ID / T8LPOL.PLAN_RTBK_CD
-- NOTE: without an ETBL_LANG_CD filter, expect 2 rows per
-- (CO_ID, PLAN_ID, RT_BK_CD) - one English, one French. That's expected
-- for this table alone; the duplication issue is in the JOIN, not here.
SELECT CO_ID, PLAN_ID, RT_BK_CD, COUNT(*) AS ROWS_PER_PLAN,
       COUNT(DISTINCT ETBL_LANG_CD) AS DISTINCT_LANGUAGES
FROM KCDWHPRC.T7PIL
WHERE CRNT_REC_IND = 'Y'
GROUP BY CO_ID, PLAN_ID, RT_BK_CD
HAVING COUNT(*) > 1;

-- TABLE: T8POL (INGENIUM policy master)
-- INFERRED KEY: CO_ID, POL_ID
-- FK OUT: joins to T8PLP (CO_ID, POL_ID) and T9PH (CO_ID, PROD_HIER_ID)
SELECT CO_ID, POL_ID, COUNT(*) AS DUP_COUNT
FROM KCDWHPRC.T8POL
WHERE CRNT_REC_IND = 'Y'
GROUP BY CO_ID, POL_ID
HAVING COUNT(*) > 1;

-- TABLE: T8PLP (INGENIUM policy premium/paid-to-date detail)
-- INFERRED KEY: CO_ID, POL_ID
-- FK IN: referenced by T8POL.CO_ID / T8POL.POL_ID
SELECT CO_ID, POL_ID, COUNT(*) AS DUP_COUNT
FROM KCDWHPRC.T8PLP
WHERE CRNT_REC_IND = 'Y'
GROUP BY CO_ID, POL_ID
HAVING COUNT(*) > 1;

-- TABLE: T9PH (INGENIUM product hierarchy)
-- INFERRED KEY: CO_ID, PROD_HIER_ID
-- FK IN: referenced by T8POL.PROD_HIER_ID
SELECT CO_ID, PROD_HIER_ID, COUNT(*) AS DUP_COUNT
FROM KCDWHPRC.T9PH
WHERE CRNT_REC_IND = 'Y'
GROUP BY CO_ID, PROD_HIER_ID
HAVING COUNT(*) > 1;


/* ----------------------------------------------------------------------
   SESSION: TT_CV  (Life70 policy master reserve / cash value)
   ---------------------------------------------------------------------- */

-- TABLE: T9LCSV
-- INFERRED KEY: CO_ID, POL_ID, CALC_AS_OF_DT
SELECT CO_ID, POL_ID, CALC_AS_OF_DT, COUNT(*) AS DUP_COUNT
FROM KCDWHPRC.T9LCSV
WHERE CRNT_REC_IND = 'Y'
GROUP BY CO_ID, POL_ID, CALC_AS_OF_DT
HAVING COUNT(*) > 1;

-- TABLE: T8SOLIFE
-- INFERRED KEY: CO_ID, POL_ID, LOAD_DT
SELECT CO_ID, POL_ID, LOAD_DT, COUNT(*) AS DUP_COUNT
FROM KCDWHPRC.T8SOLIFE
WHERE CRNT_REC_IND = 'Y'
  AND SOURCE <> 'INGENIUM' AND ANNIV_CV_AMT <> 0
GROUP BY CO_ID, POL_ID, LOAD_DT
HAVING COUNT(*) > 1;


/* ----------------------------------------------------------------------
   SESSION: TT_LCV  (life cash value messages - AAV/TAV/ANC/TNC)
   ---------------------------------------------------------------------- */

-- TABLE: T8LMSG
-- INFERRED KEY: CO_ID, POL_ID, MSG_TYP_CD, CVG_NUM, MSG_SUB_TYP_CD, EFF_DT
-- NOTE: the script's own ROW_NUMBER()...WHERE rn=1 logic de-dupes down
-- to (CO_ID, POL_ID, MSG_SUBTYP_CD) at the session-table level - this
-- check is at the RAW source level, before that de-dup happens, so
-- some duplication here may be expected/handled by the script already.
SELECT CO_ID, POL_ID, MSG_TYP_CD, CVG_NUM, MSG_SUB_TYP_CD, EFF_DT,
       COUNT(*) AS DUP_COUNT
FROM KCDWHPRC.T8LMSG
WHERE CRNT_REC_IND = 'Y'
  AND MSG_TYP_CD = '4' AND CVG_NUM = '00' AND MSG_SUB_TYP_CD <> 'TM'
GROUP BY CO_ID, POL_ID, MSG_TYP_CD, CVG_NUM, MSG_SUB_TYP_CD, EFF_DT
HAVING COUNT(*) > 1;


/* ----------------------------------------------------------------------
   SESSION: TT_PUA  (Paid-Up Additions - includes T8LPUA, our confirmed
   root-cause table)
   ---------------------------------------------------------------------- */

-- TABLE: VCPUA (view - likely Life70 PUA, non-INGENIUM)
-- INFERRED KEY: CO_ID, POL_ID, CVG_NUM, CVG_DIV_DT
SELECT CO_ID, POL_ID, CVG_NUM, CVG_DIV_DT, COUNT(*) AS DUP_COUNT
FROM KCDWHPRC.VCPUA
WHERE CRNT_REC_IND = 'Y'
GROUP BY CO_ID, POL_ID, CVG_NUM, CVG_DIV_DT
HAVING COUNT(*) > 1;

-- TABLE: T8VDV (INGENIUM PUA)
-- INFERRED KEY: CO_ID, POL_ID, CVG_NUM, CVG_DIV_DT
SELECT CO_ID, POL_ID, CVG_NUM, CVG_DIV_DT, COUNT(*) AS DUP_COUNT
FROM KCDWHPRC.T8VDV
WHERE CRNT_REC_IND = 'Y'
GROUP BY CO_ID, POL_ID, CVG_NUM, CVG_DIV_DT
HAVING COUNT(*) > 1;

-- TABLE: T8LPUA (LIFE70 PUA - CONFIRMED ROOT CAUSE TABLE)
-- CONFIRMED KEY (via SYSCAT.INDEXES, T8LPUA_UIDX1):
--   CO_ID, POL_ID, CVG_NUM, DIV_EFF_YR, L7FD_PECD
--   (CRNT_REC_IND, EFF_DT are version-control columns, correctly
--   excluded from the business key - confirmed, not a design flaw)
-- This is the exact query that found the 19 known duplicate keys.
SELECT SOURCE, CO_ID, POL_ID, CVG_NUM, DIV_EFF_DT, CVG_DIV_OPT_CD, L7FD_PECD,
       COUNT(*) AS DUP_COUNT
FROM KCDWHPRC.T8LPUA
WHERE CRNT_REC_IND = 'Y'
GROUP BY SOURCE, CO_ID, POL_ID, CVG_NUM, DIV_EFF_DT, CVG_DIV_OPT_CD, L7FD_PECD
HAVING COUNT(*) > 1;


/* ----------------------------------------------------------------------
   SESSION: TT_POLPUA  (built entirely from TT_PUA - no new source
   tables to check; if TT_PUA's three sources above are clean, this
   session table has nothing further to check at the source level)
   ---------------------------------------------------------------------- */

-- No source-level query needed here - see TT_PUA above.


/* ----------------------------------------------------------------------
   SESSION: TT_DVPCV  (INGENIUM-only dividend cash value - filtered to
   POL.SOURCE='INGENIUM' in the script, so does not inherit the
   T7PIL/LIFE70 issue)
   ---------------------------------------------------------------------- */

-- TABLE: T9DVP
-- INFERRED KEY: CO_ID, POL_ID, EFF_DT, REC_TYP_CD
-- FK OUT: joins to TT_POL filtered on SOURCE = 'INGENIUM'
SELECT CO_ID, POL_ID, EFF_DT, REC_TYP_CD, COUNT(*) AS DUP_COUNT
FROM KCDWHPRC.T9DVP
WHERE CRNT_REC_IND = 'Y' AND DERIV_VALU_FREQ_CD = 'C'
GROUP BY CO_ID, POL_ID, EFF_DT, REC_TYP_CD
HAVING COUNT(*) > 1;

-- TABLE: T9DVP_HIST
-- INFERRED KEY: CO_ID, POL_ID, EFF_DT, REC_TYP_CD
SELECT CO_ID, POL_ID, EFF_DT, REC_TYP_CD, COUNT(*) AS DUP_COUNT
FROM KCDWHPRC.T9DVP_HIST
WHERE DERIV_VALU_FREQ_CD = 'C'
GROUP BY CO_ID, POL_ID, EFF_DT, REC_TYP_CD
HAVING COUNT(*) > 1;


/* ----------------------------------------------------------------------
   SESSION: TT_CDSA  (cost-basis / dollar-sum-average transactions)
   ---------------------------------------------------------------------- */

-- TABLE: TCDSA
-- INFERRED KEY: CO_ID, POL_ID, CVG_NUM, CDA_TYP_CD, CDA_EFF_DT
-- FK OUT: joins to TDMAV via CDA_TYP_CD = DM_AV_CD
SELECT CO_ID, POL_ID, CVG_NUM, CDA_TYP_CD, CDA_EFF_DT, COUNT(*) AS DUP_COUNT
FROM KCDWHPRC.TCDSA
WHERE CDA_STAT_CD = 'A'
GROUP BY CO_ID, POL_ID, CVG_NUM, CDA_TYP_CD, CDA_EFF_DT
HAVING COUNT(*) > 1;

-- TABLE: TDMAV (lookup/reference table)
-- INFERRED KEY: DM_AV_CD, DM_AV_TBL_CD
-- FK IN: referenced by TCDSA.CDA_TYP_CD
-- A duplicate here would cause a fan-out in TCDSA's join, similar in
-- spirit to the T7PIL issue - worth checking even though not yet
-- flagged as a known problem.
SELECT DM_AV_CD, DM_AV_TBL_CD, COUNT(*) AS DUP_COUNT
FROM KCDWHPRC.TDMAV
WHERE DM_AV_TBL_CD = 'CDA-TYP-CD'
GROUP BY DM_AV_CD, DM_AV_TBL_CD
HAVING COUNT(*) > 1;


/* ----------------------------------------------------------------------
   SESSION: TT_LOAN  (policy loans - INGENIUM and LIFE70 branches)
   ---------------------------------------------------------------------- */

-- TABLE: TPOLL (INGENIUM loans)
-- INFERRED KEY: CO_ID, POL_ID, POL_LOAN_ID
SELECT CO_ID, POL_ID, POL_LOAN_ID, COUNT(*) AS DUP_COUNT
FROM KCDWHPRC.TPOLL
GROUP BY CO_ID, POL_ID, POL_LOAN_ID
HAVING COUNT(*) > 1;

-- TABLE: T8LPOLL (LIFE70 loans)
-- INFERRED KEY: CO_ID, POL_ID, EFF_DT
-- FK OUT: joins to T8D69 (CO/POLICY) for APL matching
SELECT CO_ID, POL_ID, EFF_DT, COUNT(*) AS DUP_COUNT
FROM KCDWHPRC.T8LPOLL
WHERE CRNT_REC_IND = 'Y'
  AND CO_ID IN ('01', '50') AND POL_ID > '0'
GROUP BY CO_ID, POL_ID, EFF_DT
HAVING COUNT(*) > 1;

-- TABLE: T8D69 (policy account transaction detail)
-- INFERRED KEY: CO, POLICY, ENTRDT, ACCT_TRXN_CD
-- FK IN: referenced by T8LPOLL via CO/POLICY = CO_ID/POL_ID
SELECT CO, POLICY, ENTRDT, ACCT_TRXN_CD, COUNT(*) AS DUP_COUNT
FROM KCDWHPRC.T8D69
WHERE CRNT_REC_IND = 'Y'
  AND ACCT_TRXN_CD IN ('061', '012') AND PROGCD = '3'
GROUP BY CO, POLICY, ENTRDT, ACCT_TRXN_CD
HAVING COUNT(*) > 1;


/* ----------------------------------------------------------------------
   SESSION: TT_CFLW  (Universal Life fund value cash flow - INGENIUM
   only, filtered to POL.SOURCE='INGENIUM' in the script)
   ---------------------------------------------------------------------- */

-- TABLE: TCFLW
-- INFERRED KEY: CO_ID, POL_ID, CF_EFF_DT, CF_SEQ_NUM, PREV_UPDT_TS
SELECT CO_ID, POL_ID, CF_EFF_DT, CF_SEQ_NUM, PREV_UPDT_TS, COUNT(*) AS DUP_COUNT
FROM KCDWHPRC.TCFLW
GROUP BY CO_ID, POL_ID, CF_EFF_DT, CF_SEQ_NUM, PREV_UPDT_TS
HAVING COUNT(*) > 1;


/* ============================================================================
   QUICK REFERENCE — FOREIGN KEY MAP (which tables join to which)
   ----------------------------------------------------------------------------
   TMAST ─┐
   T9DAT ─┴──> TT_RPTDT (base window, everything below depends on this)

   T8LPOL ──(PLAN_ID, PLAN_RTBK_CD)──> T7PIL          } LIFE70 branch of TT_POL
   T8POL ───(CO_ID, POL_ID)─────────> T8PLP           } INGENIUM branch
   T8POL ───(PROD_HIER_ID)──────────> T9PH            }

   T9LCSV, T8SOLIFE ──> TT_CV (independent of TT_POL)

   T8LMSG ──> TT_LCV (independent of TT_POL)

   VCPUA, T8VDV, T8LPUA ──> TT_PUA (independent of TT_POL)
   TT_PUA ──> TT_POLPUA (self-contained, no new source tables)

   T9DVP, T9DVP_HIST ──(joins TT_POL, filtered SOURCE='INGENIUM')──> TT_DVPCV

   TCDSA ──(CDA_TYP_CD = DM_AV_CD)──> TDMAV ──> TT_CDSA

   TPOLL, T8LPOLL ──(joins T8D69 by CO/POLICY)──> TT_LOAN

   TCFLW ──(joins TT_POL, filtered SOURCE='INGENIUM')──> TT_CFLW
============================================================================ */
