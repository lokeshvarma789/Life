/* ============================================================================
   DUPLICATE INVESTIGATION — WALK EVERY SESSION TABLE, DUTDM, APRIL DATA
   ----------------------------------------------------------------------------
   PURPOSE:
     Confirm the T7PIL/language-indicator duplicate hypothesis in TT_POL,
     then check every downstream session table to see whether the
     duplicate propagates or stays contained.

   RULES (per Suresh, confirmed 2026-07-10 call):
     - Run in DUTDM only. Safe, independent, no need to inform anyone.
     - DUTDM is ~1 month behind PRODDM. April data is expected to be
       present now - this is exactly the month under investigation.
     - Do NOT run the final INSERT INTO ACTUARIAL.POL_VALU. We don't
       have that statement captured anyway - see the closing note.
     - Walk every session table IN ORDER. Don't stop at the first one
       that shows duplicates - the goal is to see the whole chain.

   HOW TO USE:
     Run each numbered block top to bottom. After each session table is
     built, the duplicate-check query immediately below it tells you
     whether that table has more than one row per its intended key.
     Record the row count from each check before moving to the next
     block - that record is what tells the story across the whole file.

   WHAT WE ALREADY KNOW BEFORE RUNNING THIS:
     T7PIL is only joined inside TT_POL, and only in the LIFE70-sourced
     blocks (the Ingenium block joins T9PH instead). Checking every
     other session table's FROM/JOIN clauses that we've captured:
       - TT_CV, TT_LCV, TT_PUA, TT_POLPUA, TT_CDSA, TT_LOAN: none of
         these join back to TT_POL at all - independent of this bug.
       - TT_DVPCV, TT_CFLW: these DO join TT_POL, but filtered to
         POL.SOURCE = 'INGENIUM' only - bypassing the LIFE70/T7PIL
         branch entirely.
     So if this hypothesis is right, ONLY TT_POL should show the
     duplicate, split specifically to the non-Ingenium (LIFE70) rows.
     Steps 3 below is the one to watch closest.
============================================================================ */


/* ---- Session setup - point this connection at DUTDM ---- */
-- (!) Confirm connection/database context is DUTDM before running anything below.
-- Schema qualifiers (KCDWHPRC.*, ACTUARIAL.*) are assumed identical to PRODDM -
-- only the database/connection changes. Verify this assumption on first run.


/* ============================================================================
   STEP 1 — TT_RPTDT (base date table, sanity check only)
   Expected: exactly one row per CO_ID for the April reporting period.
   This table should never show duplicates - if it does, the whole
   investigation needs to restart here, since everything else depends
   on this window.
============================================================================ */

DROP TABLE SESSION.TT_RPTDT;

DECLARE GLOBAL TEMPORARY TABLE SESSION.TT_RPTDT AS
(
    SELECT MAST.CO_ID, MAST.BTCH_PRCES_DT AS RUN_DT,
    T9DAT.DY_NUM_YR AS RPT_YR,
    MAX (T9DAT.FINAL_DY_OF_MO_DT) AS RPT_DT,
    MIN (T9DAT.DT_VALU) AS BOM_DT
    FROM KCDWHPRC.TMAST MAST , KCDWHPRC.T9DAT T9DAT
    WHERE MAST.CO_ID= T9DAT.CO_ID
    AND T9DAT.DY_NUM_YR = year (MAST.BTCH_PRCES_DT)
    AND T9DAT.CLNDR_MO_NUM =  MONTH (MAST.BTCH_PRCES_DT)
    GROUP BY MAST.CO_ID, MAST.BTCH_PRCES_DT,T9DAT.DY_NUM_YR
)
DEFINITION ONLY on commit preserve rows not logged;

INSERT INTO SESSION.TT_RPTDT
(
    SELECT MAST.CO_ID, MAST.BTCH_PRCES_DT AS RUN_DT,
    T9DAT.DY_NUM_YR AS RPT_YR,
    MAX (T9DAT.FINAL_DY_OF_MO_DT) AS RPT_DT,
    MIN (T9DAT.DT_VALU) AS BOM_DT
    FROM KCDWHPRC.TMAST MAST , KCDWHPRC.T9DAT T9DAT
    WHERE MAST.CO_ID= T9DAT.CO_ID
    AND T9DAT.DY_NUM_YR = year (MAST.BTCH_PRCES_DT)
    AND T9DAT.CLNDR_MO_NUM =  MONTH (MAST.BTCH_PRCES_DT)-3
    GROUP BY MAST.CO_ID, MAST.BTCH_PRCES_DT,T9DAT.DY_NUM_YR
);

-- CHECK 1: should return ZERO rows
SELECT CO_ID, RPT_YR, COUNT(*) AS DUP_COUNT
FROM SESSION.TT_RPTDT
GROUP BY CO_ID, RPT_YR
HAVING COUNT(*) > 1;


/* ============================================================================
   STEP 2 — TT_POL (THE CRITICAL TABLE — T7PIL join lives here)
   This is the one where we expect to find the duplicate.
============================================================================ */

DROP TABLE SESSION.TT_POL;

DECLARE GLOBAL TEMPORARY TABLE SESSION.TT_POL AS
(
--LIFE 70 Policy data
    SELECT   POL.SOURCE,
            POL.CO_ID,
            POL.POL_ID,
            POL_BUS_CLAS_CD,
            POL.POL_CSTAT_CD, POL.POL_CSTAT_TXT, POL.POL_INS_TYP_CD,
            PH.INS_SUB_TYP_CD, PH.INS_SUB_TYP_TXT,
            CASE
                WHEN POL_BUS_CLAS_CD = 'A' AND POL.POL_INS_TYP_CD ='N' THEN 'FPA'
                ELSE POL_INS_TYP_TXT
                END AS INS_TYP_TXT,
            POL.PLAN_ID AS POL_PLAN_ID,
            POL_ISS_EFF_DT,
            POL.POL_ISS_LOC_CD,POL.CLI_CRNT_LOC_CD, POL.ELIG_MB_CO_NMBR, POL.POL_ORIG_TXT,
            CASE WHEN POL_CEAS_DT IS NULL THEN POL.POL_STAT_CHNG_DT ELSE POL.POL_CEAS_DT END AS POL_CEAS_DT,
            POL.POL_MPREM_AMT,
            POL.POL_BILL_MODE_CD,
            POL.POL_BILL_TYP_TXT,
            POL_MPREM_AMT*12/DECIMAL (POL_BILL_MODE_CD) AS POL_GRS_APREM_AMT,
            POL_PD_TO_DT,
            POL_ISS_EFF_DT + (TIMESTAMPDIFF (256, CHAR (timestamp (RPT_DT) - timestamp (POL_ISS_EFF_DT))))YEAR
                AS PREV_ANNV_DT,
            POL.POL_STAT_CHNG_DT,
            DT.RPT_DT AS SNAP_SHOT_DT
    FROM     KCDWHPRC.T8LPOL POL,
            SESSION.TT_RPTDT DT,
            KCDWHPRC.T7PIL PH
    WHERE    POL.CO_ID = DT.CO_ID
    AND      POL.CRNT_REC_IND='Y'
    AND      POL.CO_ID = PH.CO_ID
    AND      POL.PLAN_ID = PH.PLAN_ID
    AND   POL.PLAN_RTBK_CD = PH.RT_BK_CD
    AND      PH.CRNT_REC_IND ='Y'
AND PH.ETBL_LANG_CD='E'
AND POL.POL_CEAS_DT >= DT.RPT_DT - 1 YEAR --excluding old terminated policies

)
DEFINITION ONLY on commit preserve rows not logged;

INSERT INTO SESSION.TT_POL
(
--INGENIUM policy data
    SELECT   'INGENIUM' AS SOURCE,
            POL.CO_ID,
            POL.POL_ID,
            POL_BUS_CLAS_CD,
            POL.POL_CSTAT_CD, POL.POL_CSTAT_TXT, POL.POL_INS_TYP_CD,
            PH.INS_SUB_TYP_CD, PH.INS_SUBTP_DESC_TXT,
            POL_INS_TYP_TXT ,
            POL.PLAN_ID AS POL_PLAN_ID,
            POL_ISS_EFF_DT,
            POL.POL_ISS_LOC_CD,POL.CLI_CRNT_LOC_CD, POL.ELIG_MB_CO_NMBR, POL.POL_ORIG_TXT,
            CASE WHEN POL_CEAS_DT IS NULL THEN POL.POL_STAT_CHNG_DT ELSE POL.POL_CEAS_DT END AS POL_CEAS_DT,
            POL_MPREM_AMT,
            POL_BILL_MODE_CD,
            POL_BILL_TYP_TXT,
            POL_GRS_APREM_AMT,
            PLP.POL_PD_TO_DT,
            POL_ISS_EFF_DT + (TIMESTAMPDIFF (256, CHAR (timestamp (RPT_DT) - timestamp (POL_ISS_EFF_DT))))YEAR
                AS PREV_ANNV_DT,
            POL.POL_STAT_CHNG_DT,
            DT.RPT_DT AS SNAP_SHOT_DT
    FROM     KCDWHPRC.T8POL POL,
            SESSION.TT_RPTDT DT,
            KCDWHPRC.T8PLP PLP, -- no dups
            KCDWHPRC.T9PH PH
    WHERE    POL.CO_ID = DT.CO_ID
    AND      POL.CRNT_REC_IND='Y'
    AND      POL.CO_ID= PLP.CO_ID
    AND      POL.POL_ID= PLP.POL_ID
    AND      PLP.CRNT_REC_IND='Y'
    AND POL.CO_ID= PH.CO_ID
    AND POL.PROD_HIER_ID= PH.PROD_HIER_ID
    AND PH.CRNT_REC_IND='Y'
    AND POL.POL_BUS_CLAS_CD = 'L' --LIFE ONLY. INGENIUM annuity value is in IA_VALU table
AND POL.POL_CEAS_DT >= DT.RPT_DT - 1 YEAR --excluding old terminated policies

);

-- NOTE: the original script has a THIRD insert here that is an exact
-- duplicate of the first LIFE70 block (same T7PIL join, same filter).
-- Deliberately EXCLUDED from this investigation run - if it's a real
-- copy-paste duplicate in the production script, including it here
-- would double-count the LIFE70 rows and give us a false duplicate
-- signal. Flagged for separate confirmation with whoever owns the
-- original script - was this third INSERT meant to be there at all?

-- CHECK 2A: overall duplicate check - all sources combined
SELECT SOURCE, CO_ID, POL_ID, COUNT(*) AS DUP_COUNT
FROM SESSION.TT_POL
GROUP BY SOURCE, CO_ID, POL_ID
HAVING COUNT(*) > 1
ORDER BY DUP_COUNT DESC;

-- CHECK 2B: THE KEY TEST - isolate LIFE70 (non-Ingenium) rows only
-- This is where the T7PIL/language-indicator duplicate should show up
-- if the hypothesis is correct. If CHECK 2B returns rows and CHECK 2C
-- (below) returns none, that CONFIRMS the fix (ETBL_LANG_CD='E') is
-- necessary and the LIFE70/T7PIL join is the sole cause.
SELECT SOURCE, CO_ID, POL_ID, COUNT(*) AS DUP_COUNT
FROM SESSION.TT_POL
WHERE SOURCE <> 'INGENIUM'
GROUP BY SOURCE, CO_ID, POL_ID
HAVING COUNT(*) > 1
ORDER BY DUP_COUNT DESC;

-- CHECK 2C: confirm Ingenium rows are clean (different join, no T7PIL)
-- Expected: ZERO rows. If this ALSO shows duplicates, there's a second,
-- unrelated issue in the T9PH join that needs separate investigation.
SELECT SOURCE, CO_ID, POL_ID, COUNT(*) AS DUP_COUNT
FROM SESSION.TT_POL
WHERE SOURCE = 'INGENIUM'
GROUP BY SOURCE, CO_ID, POL_ID
HAVING COUNT(*) > 1
ORDER BY DUP_COUNT DESC;

-- CHECK 2D (diagnostic, only run if 2B shows duplicates): pull the
-- actual duplicated rows so you can SEE the ETBL_LANG_CD values side by
-- side and visually confirm the English/French pairing.
SELECT
    POL.CO_ID, POL.POL_ID, POL.PLAN_ID AS POL_PLAN_ID,
    PH.PLAN_ID, PH.RT_BK_CD, PH.ETBL_LANG_CD, PH.CRNT_REC_IND
FROM KCDWHPRC.T8LPOL POL
INNER JOIN SESSION.TT_RPTDT DT ON POL.CO_ID = DT.CO_ID
INNER JOIN KCDWHPRC.T7PIL PH
    ON POL.CO_ID = PH.CO_ID
    AND POL.PLAN_ID = PH.PLAN_ID
    AND POL.PLAN_RTBK_CD = PH.RT_BK_CD
WHERE POL.CRNT_REC_IND = 'Y'
  AND PH.CRNT_REC_IND = 'Y'
  AND POL.CO_ID || '-' || POL.POL_ID IN (
      -- (!) paste one duplicated CO_ID/POL_ID pair from CHECK 2B here, e.g.:
      -- '50-1234567'
      'PASTE_CO_ID_POL_ID_HERE'
  )
ORDER BY POL.CO_ID, POL.POL_ID, PH.ETBL_LANG_CD;


/* ============================================================================
   STEP 3 — TT_CV (independent of TT_POL - not expected to inherit the bug)
============================================================================ */

DROP TABLE SESSION.TT_CV;

DECLARE GLOBAL TEMPORARY TABLE SESSION.TT_CV AS
(
    SELECT   'L70PMR' AS SOURCE,
            CSV.CO_ID,
            CSV.POL_ID,
            CALC_AS_OF_DT,
            CSV.CV_AMT,
            CSV_AMT,
            CSV.PUA_CV_AMT
    FROM     KCDWHPRC.T9LCSV CSV
    INNER JOIN SESSION.TT_RPTDT DT
    ON CSV.CO_ID = DT.CO_ID
    AND CSV.CALC_AS_OF_DT > DT.RPT_DT - 1 YEAR
    AND CSV.CRNT_REC_IND='Y'
)
DEFINITION ONLY on commit preserve rows not logged;

INSERT INTO SESSION.TT_CV
(
    SELECT   'L70PMR' AS SOURCE,
            CSV.CO_ID,
            CSV.POL_ID,
            CALC_AS_OF_DT,
            CSV.CV_AMT,
            CSV_AMT,
            CSV.PUA_CV_AMT
    FROM     KCDWHPRC.T9LCSV CSV
    INNER JOIN SESSION.TT_RPTDT DT
    ON CSV.CO_ID = DT.CO_ID
    AND CSV.CALC_AS_OF_DT > DT.RPT_DT - 1 YEAR
    AND CSV.CRNT_REC_IND='Y'
);

INSERT INTO SESSION.TT_CV
(
--LIFE CASH VALUES AT ANNIVERSARY
    SELECT   SO.SOURCE,
            SO.CO_ID,
            SO.POL_ID,
            LOAD_DT,
            ANNIV_CV_AMT,
            SO.POL_CSV_AMT,
            SO.PUA_CV_AMT
    FROM     KCDWHPRC.T8SOLIFE SO
    INNER JOIN SESSION.TT_RPTDT DT
    ON      SO.CO_ID = DT.CO_ID
    AND     CRNT_REC_IND = 'Y'
    AND     SOURCE <> 'INGENIUM'
    AND     SO.ANNIV_CV_AMT <> 0
    AND     SO.LOAD_DT > DT.RPT_DT - 1 YEAR

    LEFT OUTER JOIN SESSION.TT_CV CV
    ON SO.CO_ID= CV.CO_ID
    AND SO.POL_ID= CV.POL_ID
    AND CV.CALC_AS_OF_DT > DT.RPT_DT - 1 year
    WHERE CV.CV_AMT is NULL
);

-- CHECK 3: expected ZERO (no dependency on TT_POL/T7PIL)
SELECT SOURCE, CO_ID, POL_ID, CALC_AS_OF_DT, COUNT(*) AS DUP_COUNT
FROM SESSION.TT_CV
GROUP BY SOURCE, CO_ID, POL_ID, CALC_AS_OF_DT
HAVING COUNT(*) > 1
ORDER BY DUP_COUNT DESC;


/* ============================================================================
   STEP 4 — TT_LCV (independent of TT_POL)
============================================================================ */

DROP TABLE SESSION.TT_LCV;

DECLARE GLOBAL TEMPORARY TABLE SESSION.TT_LCV AS
(
    SELECT SOURCE, CO_ID, POL_ID, EFF_DT, MSG_DT, MSG_XPRY_DT, MSG_SUBTYP_CD, MSG_TXT, POL_CV_QT
    FROM (
    SELECT ROW_NUMBER() OVER (PARTITION BY co_id,pol_id,msg_subtyp_cd ORDER BY eff_dt desc,msg_xpry_dt desc) AS rn,
      a.*
      FROM (
            SELECT   MSG.SOURCE, MSG.CO_ID, MSG.POL_ID, EFF_DT, MSG_DT,
                    CASE WHEN MSG_XPRY_DT IS NULL THEN '9999-12-31' ELSE MSG_XPRY_DT END AS MSG_XPRY_DT,
                    CASE WHEN SUBSTR (MSG_TXT,1,1) = '$' THEN 'AAV'
                        WHEN SUBSTR (MSG_TXT,1,1) = '#' THEN 'TAV'
                        WHEN SUBSTR (MSG_TXT,1,1) = '@' THEN 'ANC'
                        WHEN SUBSTR (MSG_TXT,1,1) = '.' THEN 'TNC'
                        END AS MSG_SUBTYP_CD,
                    MSG_TXT,
                    TRIM (LEADING '0' FROM (SUBSTR (MSG_TXT,2,10)))AS POL_CV_QT
            FROM     SESSION.TT_RPTDT DT, KCDWHPRC.T8LMSG MSG
            WHERE    MSG.CO_ID = DT.CO_ID
            AND      MSG.CO_ID ='50'
            AND      CRNT_REC_IND='Y'
            AND      MSG_TYP_CD='4'
            AND      CVG_NUM='00'
            AND      MSG_SUB_TYP_CD<>'TM'
            AND      SUBSTR (MSG_TXT,1,1) IN ('$', '#', '@', '.')
            AND SUBSTR (MSG_TXT,2,10)<> '0000000000'
            AND SUBSTR (MSG_TXT,11,1) BETWEEN '0' AND '9'
              AND ( MSG.MSG_XPRY_DT > DT.RPT_DT - 1 YEAR OR MSG.MSG_XPRY_DT IS NULL)
        ) a)
    WHERE rn = 1
)
DEFINITION ONLY on commit preserve rows not logged;

INSERT INTO SESSION.TT_LCV
(
    SELECT SOURCE, CO_ID, POL_ID, EFF_DT, MSG_DT, MSG_XPRY_DT, MSG_SUBTYP_CD, MSG_TXT, POL_CV_QT
    FROM (
    SELECT ROW_NUMBER() OVER (PARTITION BY co_id,pol_id,msg_subtyp_cd ORDER BY eff_dt desc,msg_xpry_dt desc) AS rn,
      a.*
      FROM (
            SELECT   MSG.SOURCE, MSG.CO_ID, MSG.POL_ID, EFF_DT, MSG_DT,
                    CASE WHEN MSG_XPRY_DT IS NULL THEN '9999-12-31' ELSE MSG_XPRY_DT END AS MSG_XPRY_DT,
                    CASE WHEN SUBSTR (MSG_TXT,1,1) = '$' THEN 'AAV'
                        WHEN SUBSTR (MSG_TXT,1,1) = '#' THEN 'TAV'
                        WHEN SUBSTR (MSG_TXT,1,1) = '@' THEN 'ANC'
                        WHEN SUBSTR (MSG_TXT,1,1) = '.' THEN 'TNC'
                        END AS MSG_SUBTYP_CD,
                    MSG_TXT,
                    TRIM (LEADING '0' FROM (SUBSTR (MSG_TXT,2,10)))AS POL_CV_QT
            FROM     SESSION.TT_RPTDT DT, KCDWHPRC.T8LMSG MSG
            WHERE    MSG.CO_ID = DT.CO_ID
            AND      MSG.CO_ID ='50'
            AND      CRNT_REC_IND='Y'
            AND      MSG_TYP_CD='4'
            AND      CVG_NUM='00'
            AND      MSG_SUB_TYP_CD<>'TM'
            AND      SUBSTR (MSG_TXT,1,1) IN ('$', '#', '@', '.')
            AND SUBSTR (MSG_TXT,2,10)<> '0000000000'
            AND SUBSTR (MSG_TXT,11,1) BETWEEN '0' AND '9'
              AND ( MSG.MSG_XPRY_DT > DT.RPT_DT - 1 YEAR OR MSG.MSG_XPRY_DT IS NULL)
        ) a)
    WHERE rn = 1
);

-- CHECK 4: expected ZERO (rn=1 filter already de-dupes by design)
SELECT CO_ID, POL_ID, MSG_SUBTYP_CD, COUNT(*) AS DUP_COUNT
FROM SESSION.TT_LCV
GROUP BY CO_ID, POL_ID, MSG_SUBTYP_CD
HAVING COUNT(*) > 1
ORDER BY DUP_COUNT DESC;


/* ============================================================================
   STEP 5 — TT_PUA (independent of TT_POL)
============================================================================ */

DROP TABLE SESSION.TT_PUA;

DECLARE GLOBAL TEMPORARY TABLE SESSION.TT_PUA AS
(
    select SOURCE, PUA.CO_ID, POL_ID, CVG_NUM, CVG_DIV_DT,
            PUA.CVG_DIV_OPT_CD, PUA.CVG_DIV_OPT_TXT,
            CVG_DOD_ACUM_AMT, PUA_LTD_FACE_AMT, INSID_PUA_LTD_AMT,
            CASE
                WHEN PUA.SOURCE<>'INGENIUM' AND L7FD_PECD = '0' THEN COALESCE (PUA_LTD_FACE_AMT,0) + COALESCE (INSID_PUA_LTD_AMT,0)
                WHEN PUA.SOURCE<>'INGENIUM' AND L7FD_PECD IN ('I','D','C') THEN PUA_CLR_FACE_AMT
                WHEN PUA.SOURCE = 'INGENIUM' THEN COALESCE (PUA_LTD_FACE_AMT,0) + COALESCE (INSID_PUA_LTD_AMT,0)
                ELSE 0
            END AS TOT_PUA_AMT,
            PUA_CLR_FACE_AMT, L7FD_PECD, L7FD_PECD_TXT
    from kcdwhPRC.vcpua  PUA, SESSION.TT_RPTDT DT
    WHERE   PUA.CO_ID = DT.CO_ID
    AND PUA.CVG_DIV_DT >= DT.RPT_DT - 1 year
    AND     CRNT_REC_IND = 'Y'
)
DEFINITION ONLY on commit preserve rows not logged;

INSERT INTO SESSION.TT_PUA
(
    select 'INGENIUM' AS SOURCE, PUA.CO_ID, POL_ID, CVG_NUM, CVG_DIV_DT,
            PUA.CVG_DIV_OPT_CD, PUA.CVG_DIV_OPT_TXT,
            CVG_DOD_ACUM_AMT, PUA_LTD_FACE_AMT, INSID_PUA_LTD_AMT,
            COALESCE (PUA_LTD_FACE_AMT,0) + COALESCE (INSID_PUA_LTD_AMT,0) AS TOT_PUA_AMT,
            PUA_CLR_FACE_AMT, NULL AS L7FD_PECD, NULL AS L7FD_PECD_TXT
    from   kcdwhPRC.T8VDV PUA, SESSION.TT_RPTDT DT
    WHERE   PUA.CO_ID = DT.CO_ID
    AND PUA.CVG_DIV_DT >= DT.RPT_DT - 1 year
    AND CRNT_REC_IND = 'Y'
);

INSERT INTO SESSION.TT_PUA
(
    select SOURCE, PUA.CO_ID, POL_ID, CVG_NUM, PUA.DIV_EFF_DT,
            PUA.CVG_DIV_OPT_CD, PUA.CVG_DIV_OPT_TXT,
            CVG_DOD_ACUM_AMT, PUA_LTD_FACE_AMT, INSID_PUA_LTD_AMT,
            CASE
                WHEN L7FD_PECD = '0' THEN COALESCE (PUA_LTD_FACE_AMT,0) + COALESCE (INSID_PUA_LTD_AMT,0)
                WHEN L7FD_PECD IN ('I','D','C') THEN PUA_CLR_FACE_AMT
                ELSE 0
            END AS TOT_PUA_AMT,
            PUA_CLR_FACE_AMT, L7FD_PECD, L7FD_PECD_TXT
    from   kcdwhPRC.T8LPUA PUA, SESSION.TT_RPTDT DT
    WHERE   PUA.CO_ID = DT.CO_ID
    AND PUA.DIV_EFF_DT >= DT.RPT_DT - 1 year
     AND PUA.L7FD_PECD='0'
    AND CRNT_REC_IND = 'Y'
);

-- CHECK 5: expected ZERO
SELECT SOURCE, CO_ID, POL_ID, CVG_NUM, CVG_DIV_DT, COUNT(*) AS DUP_COUNT
FROM SESSION.TT_PUA
GROUP BY SOURCE, CO_ID, POL_ID, CVG_NUM, CVG_DIV_DT
HAVING COUNT(*) > 1
ORDER BY DUP_COUNT DESC;


/* ============================================================================
   STEP 6 — TT_POLPUA (built from TT_PUA only, independent of TT_POL)
============================================================================ */

DROP TABLE SESSION.TT_POLPUA;

DECLARE GLOBAL TEMPORARY TABLE SESSION.TT_POLPUA AS
(
    SELECT PUA.SOURCE , PUA.CO_ID , PUA.POL_ID , PUA.CVG_DIV_DT ,
    L7FD_PECD, NULL AS TRM_RSN_CD, L7FD_PECD_TXT,
    SUM (PUA.CVG_DOD_ACUM_AMT) DOD_ACUM_AMT,
    SUM (PUA_LTD_FACE_AMT) PUA_LTD_FACE_AMT,
    SUM (INSID_PUA_LTD_AMT) INSID_PUA_LTD_AMT,
    SUM (TOT_PUA_AMT) TOT_PUA_AMT,
    SUM (PUA_CLR_FACE_AMT)PUA_CLR_FACE_AMT
    FROM  SESSION.TT_PUA PUA
    INNER JOIN
    (
        SELECT SOURCE , CO_ID , POL_ID , CVG_NUM ,
        COUNT (CVG_NUM)AS REC_COUNT,
        MAX (CVG_DIV_DT) AS MAX_DIV_DT,
        MIN (CVG_DIV_DT) AS MIN_DIV_DT
        FROM  SESSION.TT_PUA
        GROUP BY  SOURCE , CO_ID , POL_ID , CVG_NUM
    )MAXPUA
    ON PUA.SOURCE= MAXPUA.SOURCE
    AND PUA.CO_ID= MAXPUA.CO_ID
    AND PUA.POL_ID= MAXPUA.POL_ID
    AND PUA.CVG_NUM= MAXPUA.CVG_NUM
    AND PUA.CVG_DIV_DT= MAXPUA.MAX_DIV_DT
    GROUP BY  PUA.SOURCE , PUA.CO_ID , PUA.POL_ID , PUA.CVG_DIV_DT ,
    L7FD_PECD, L7FD_PECD_TXT
)
DEFINITION ONLY on commit preserve rows not logged;

INSERT INTO SESSION.TT_POLPUA
(
    SELECT PUA.SOURCE , PUA.CO_ID , PUA.POL_ID , PUA.CVG_DIV_DT ,
    L7FD_PECD,
    CASE WHEN L7FD_PECD = 'C' THEN 'E'
        WHEN L7FD_PECD = 'D' THEN 'F'
        WHEN L7FD_PECD = 'E' THEN 'H'
        WHEN L7FD_PECD = 'G' THEN 'X'
        WHEN L7FD_PECD = 'I' THEN 'D'
        WHEN L7FD_PECD = 'A' THEN 'A'
        WHEN L7FD_PECD = 'B' THEN 'B'
            END AS TRM_RSN_CD,
            L7FD_PECD_TXT,
    SUM (PUA.CVG_DOD_ACUM_AMT) DOD_ACUM_AMT,
    SUM (PUA_LTD_FACE_AMT) PUA_LTD_FACE_AMT,
    SUM (INSID_PUA_LTD_AMT) INSID_PUA_LTD_AMT,
    SUM (TOT_PUA_AMT) TOT_PUA_AMT,
    SUM (PUA_CLR_FACE_AMT)PUA_CLR_FACE_AMT
    FROM  SESSION.TT_PUA PUA
    INNER JOIN
    (
        SELECT SOURCE , CO_ID , POL_ID , CVG_NUM ,
        COUNT (CVG_NUM)AS REC_COUNT,
        MAX (CVG_DIV_DT) AS MAX_DIV_DT,
        MIN (CVG_DIV_DT) AS MIN_DIV_DT
        FROM  SESSION.TT_PUA
        GROUP BY  SOURCE , CO_ID , POL_ID , CVG_NUM
    )MAXPUA
    ON PUA.SOURCE= MAXPUA.SOURCE
    AND PUA.CO_ID= MAXPUA.CO_ID
    AND PUA.POL_ID= MAXPUA.POL_ID
    AND PUA.CVG_NUM= MAXPUA.CVG_NUM
    AND PUA.CVG_DIV_DT= MAXPUA.MAX_DIV_DT
    GROUP BY  PUA.SOURCE , PUA.CO_ID , PUA.POL_ID , PUA.CVG_DIV_DT ,
    L7FD_PECD, L7FD_PECD_TXT
);

-- CHECK 6A: exact-grain check (matches the INSERT's own GROUP BY)
SELECT SOURCE, CO_ID, POL_ID, CVG_DIV_DT, L7FD_PECD, COUNT(*) AS DUP_COUNT
FROM SESSION.TT_POLPUA
GROUP BY SOURCE, CO_ID, POL_ID, CVG_DIV_DT, L7FD_PECD
HAVING COUNT(*) > 1
ORDER BY DUP_COUNT DESC;

-- CHECK 6B: looser check matching the table's OWN index (SOURCE, CO_ID, POL_ID)
-- This one MAY legitimately show >1 row per policy if a policy has
-- multiple coverages with different CVG_DIV_DT values - not necessarily
-- a bug. Included for completeness; interpret with that caveat.
SELECT SOURCE, CO_ID, POL_ID, COUNT(*) AS ROW_COUNT
FROM SESSION.TT_POLPUA
GROUP BY SOURCE, CO_ID, POL_ID
HAVING COUNT(*) > 1
ORDER BY ROW_COUNT DESC;


/* ============================================================================
   STEP 7 — TT_DVPCV (joins TT_POL, but filtered to SOURCE='INGENIUM' only -
   should NOT inherit the LIFE70/T7PIL duplicate, verify anyway)
============================================================================ */

DROP TABLE SESSION.TT_DVPCV;

DECLARE GLOBAL TEMPORARY TABLE SESSION.TT_DVPCV AS
(
    SELECT CO_ID, POL_ID, CALC_AS_OF_DT,
    POL_CSV_AMT, POL_BASE_CV_AMT, PUA_CV_AMT, INSID_PUA_CV_AMT,
    PUA_MAXCV_AMT, DOD_MAXWTHD_AMT
    FROM (
        SELECT ROW_NUMBER() OVER (PARTITION BY co_id,pol_id ORDER BY calc_as_of_dt desc) AS rn,
        a.*
        FROM
        (
            SELECT    CO_ID, POL_ID, CALC_AS_OF_DT,
                    SUM (POL_CSV_AMT) POL_CSV_AMT,
                    SUM (POL_BASE_CV_AMT) POL_BASE_CV_AMT,
                    SUM (PUA_CV_AMT) PUA_CV_AMT,
                    SUM (INSID_PUA_CV_AMT) INSID_PUA_CV_AMT,
                    SUM (PUA_MAXCV_AMT) PUA_MAXCV_AMT,
                    SUM (DOD_MAXWTHD_AMT) DOD_MAXWTHD_AMT
            FROM
            (
                SELECT   DVP.CO_ID, DVP.POL_ID, DVP.REC_TYP_CD, CALC_AS_OF_DT,
                        CASE WHEN DVP.REC_TYP_CD = 'P' THEN DVP.DV_POL_CSV_AMT ELSE 0 END AS POL_CSV_AMT,
                        CASE WHEN DVP.REC_TYP_CD = 'P' THEN DVP.DV_POL_BASE_CV_AMT ELSE 0 END AS POL_BASE_CV_AMT,
                        CASE WHEN DVP.REC_TYP_CD = 'C' THEN DVP.DV_CVG_PUA_CV_AMT ELSE 0 END  AS PUA_CV_AMT,
                        CASE WHEN DVP.REC_TYP_CD = 'C' THEN DVP.DV_INSID_PUA_CV_AMT ELSE 0 END AS INSID_PUA_CV_AMT,
                        CASE WHEN DVP.REC_TYP_CD = 'C' THEN DVP.DV_POL_PUA_MAXCV_AMT ELSE 0 END AS PUA_MAXCV_AMT,
                        CASE WHEN DVP.REC_TYP_CD = 'C' THEN DVP.DV_POL_DOD_MAXWTHD_AMT ELSE 0 END AS DOD_MAXWTHD_AMT
                FROM     KCDWHPRC.T9DVP DVP
                INNER JOIN SESSION.TT_POL POL
                ON POL.SOURCE = 'INGENIUM'
                AND POL.CO_ID = DVP.CO_ID
                AND POL.POL_ID = DVP.POL_ID
                AND DVP.CRNT_REC_IND='Y'
                INNER JOIN SESSION.TT_RPTDT DT
                ON DT.CO_ID =DVP.CO_ID
                AND DVP.EFF_DT > DT.RPT_DT - 1 year
                AND DVP.EFF_DT <= DT.RPT_DT
                AND DVP.DERIV_VALU_FREQ_CD = 'C'
                UNION
                SELECT   DVP.CO_ID, DVP.POL_ID, DVP.REC_TYP_CD, CALC_AS_OF_DT,
                        CASE WHEN DVP.REC_TYP_CD = 'P' THEN DVP.DV_POL_CSV_AMT ELSE 0 END AS POL_CSV_AMT,
                        CASE WHEN DVP.REC_TYP_CD = 'P' THEN DVP.DV_POL_BASE_CV_AMT ELSE 0 END AS POL_BASE_CV_AMT,
                        CASE WHEN DVP.REC_TYP_CD = 'C' THEN DVP.DV_CVG_PUA_CV_AMT ELSE 0 END  AS PUA_CV_AMT,
                        CASE WHEN DVP.REC_TYP_CD = 'C' THEN DVP.DV_INSID_PUA_CV_AMT ELSE 0 END AS INSID_PUA_CV_AMT,
                        CASE WHEN DVP.REC_TYP_CD = 'C' THEN DVP.DV_POL_PUA_MAXCV_AMT ELSE 0 END AS PUA_MAXCV_AMT,
                        CASE WHEN DVP.REC_TYP_CD = 'C' THEN DVP.DV_POL_DOD_MAXWTHD_AMT ELSE 0 END AS DOD_MAXWTHD_AMT
                FROM KCDWHPRC.T9DVP_HIST DVP
                INNER JOIN SESSION.TT_POL POL
                ON POL.SOURCE = 'INGENIUM'
                AND POL.CO_ID = DVP.CO_ID
                AND POL.POL_ID = DVP.POL_ID
                INNER JOIN SESSION.TT_RPTDT DT
                ON DT.CO_ID =DVP.CO_ID
                AND DVP.EFF_DT > DT.RPT_DT - 1 year
                AND DVP.EFF_DT <= DT.RPT_DT
                AND DVP.DERIV_VALU_FREQ_CD = 'C'
            )
                    GROUP BY CO_ID, POL_ID, CALC_AS_OF_DT) A
        )
        WHERE rn = 1
)
DEFINITION ONLY on commit preserve rows not logged;

INSERT INTO SESSION.TT_DVPCV
(
    SELECT CO_ID, POL_ID, CALC_AS_OF_DT,
    POL_CSV_AMT, POL_BASE_CV_AMT, PUA_CV_AMT, INSID_PUA_CV_AMT,
    PUA_MAXCV_AMT, DOD_MAXWTHD_AMT
    FROM (
        SELECT ROW_NUMBER() OVER (PARTITION BY co_id,pol_id ORDER BY calc_as_of_dt desc) AS rn,
        a.*
        FROM
        (
            SELECT    CO_ID, POL_ID, CALC_AS_OF_DT,
                    SUM (POL_CSV_AMT) POL_CSV_AMT,
                    SUM (POL_BASE_CV_AMT) POL_BASE_CV_AMT,
                    SUM (PUA_CV_AMT) PUA_CV_AMT,
                    SUM (INSID_PUA_CV_AMT) INSID_PUA_CV_AMT,
                    SUM (PUA_MAXCV_AMT) PUA_MAXCV_AMT,
                    SUM (DOD_MAXWTHD_AMT) DOD_MAXWTHD_AMT
            FROM
            (
                SELECT   DVP.CO_ID, DVP.POL_ID, DVP.REC_TYP_CD, CALC_AS_OF_DT,
                        CASE WHEN DVP.REC_TYP_CD = 'P' THEN DVP.DV_POL_CSV_AMT ELSE 0 END AS POL_CSV_AMT,
                        CASE WHEN DVP.REC_TYP_CD = 'P' THEN DVP.DV_POL_BASE_CV_AMT ELSE 0 END AS POL_BASE_CV_AMT,
                        CASE WHEN DVP.REC_TYP_CD = 'C' THEN DVP.DV_CVG_PUA_CV_AMT ELSE 0 END  AS PUA_CV_AMT,
                        CASE WHEN DVP.REC_TYP_CD = 'C' THEN DVP.DV_INSID_PUA_CV_AMT ELSE 0 END AS INSID_PUA_CV_AMT,
                        CASE WHEN DVP.REC_TYP_CD = 'C' THEN DVP.DV_POL_PUA_MAXCV_AMT ELSE 0 END AS PUA_MAXCV_AMT,
                        CASE WHEN DVP.REC_TYP_CD = 'C' THEN DVP.DV_POL_DOD_MAXWTHD_AMT ELSE 0 END AS DOD_MAXWTHD_AMT
                FROM     KCDWHPRC.T9DVP DVP
                INNER JOIN SESSION.TT_POL POL
                ON POL.SOURCE = 'INGENIUM'
                AND POL.CO_ID = DVP.CO_ID
                AND POL.POL_ID = DVP.POL_ID
                AND DVP.CO_ID = '50'
                AND DVP.CRNT_REC_IND='Y'
                INNER JOIN SESSION.TT_RPTDT DT
                ON DT.CO_ID =DVP.CO_ID
                AND DVP.EFF_DT > DT.RPT_DT - 1 year
                AND DVP.EFF_DT <= DT.RPT_DT
                AND DVP.DERIV_VALU_FREQ_CD = 'C'
                UNION
                SELECT   DVP.CO_ID, DVP.POL_ID, DVP.REC_TYP_CD, CALC_AS_OF_DT,
                        CASE WHEN DVP.REC_TYP_CD = 'P' THEN DVP.DV_POL_CSV_AMT ELSE 0 END AS POL_CSV_AMT,
                        CASE WHEN DVP.REC_TYP_CD = 'P' THEN DVP.DV_POL_BASE_CV_AMT ELSE 0 END AS POL_BASE_CV_AMT,
                        CASE WHEN DVP.REC_TYP_CD = 'C' THEN DVP.DV_CVG_PUA_CV_AMT ELSE 0 END  AS PUA_CV_AMT,
                        CASE WHEN DVP.REC_TYP_CD = 'C' THEN DVP.DV_INSID_PUA_CV_AMT ELSE 0 END AS INSID_PUA_CV_AMT,
                        CASE WHEN DVP.REC_TYP_CD = 'C' THEN DVP.DV_POL_PUA_MAXCV_AMT ELSE 0 END AS PUA_MAXCV_AMT,
                        CASE WHEN DVP.REC_TYP_CD = 'C' THEN DVP.DV_POL_DOD_MAXWTHD_AMT ELSE 0 END AS DOD_MAXWTHD_AMT
                FROM KCDWHPRC.T9DVP_HIST DVP
                INNER JOIN SESSION.TT_POL POL
                ON POL.SOURCE = 'INGENIUM'
                AND POL.CO_ID = DVP.CO_ID
                AND POL.POL_ID = DVP.POL_ID
                AND DVP.CO_ID = '01'
                AND DVP.CRNT_REC_IND='Y'
                INNER JOIN SESSION.TT_RPTDT DT
                ON DT.CO_ID =DVP.CO_ID
                AND DVP.EFF_DT > DT.RPT_DT - 1 year
                AND DVP.EFF_DT <= DT.RPT_DT
                AND DVP.DERIV_VALU_FREQ_CD = 'C'
            )
                    GROUP BY CO_ID, POL_ID, CALC_AS_OF_DT) A
        )
        WHERE rn = 1
);

-- CHECK 7: expected ZERO (rn=1 filter, and INGENIUM-only source)
SELECT CO_ID, POL_ID, COUNT(*) AS DUP_COUNT
FROM SESSION.TT_DVPCV
GROUP BY CO_ID, POL_ID
HAVING COUNT(*) > 1
ORDER BY DUP_COUNT DESC;


/* ============================================================================
   STEP 8 — TT_CDSA (independent of TT_POL)
============================================================================ */

DROP TABLE SESSION.TT_CDSA ;

DECLARE GLOBAL TEMPORARY TABLE SESSION.TT_CDSA AS
(
    SELECT   A.CO_ID, POL_ID, CVG_NUM, A.CDA_TYP_CD,
            D.DM_AV_MODEL_TXT AS CDA_TYP_TXT,
            SUM (CDA_TOT_TRXN_AMT) AS TOT_TRXN_AMT
    FROM     KCDWHPRC.TCDSA A, KCDWHPRC.TDMAV D, SESSION.TT_RPTDT DT
    WHERE    A.CO_ID = DT.CO_ID
    AND      A.CDA_TYP_CD= D.DM_AV_CD
    AND      D.DM_AV_TBL_CD='CDA-TYP-CD'
    AND      CDA_EFF_DT > DT.RPT_DT - 1 YEAR
    AND      CDA_STAT_CD = 'A'
    GROUP BY A.CO_ID, POL_ID, CVG_NUM,A.CDA_TYP_CD, D.DM_AV_MODEL_TXT
)
DEFINITION ONLY on commit preserve rows not logged;

INSERT INTO SESSION.TT_CDSA
(
    SELECT   A.CO_ID, POL_ID, CVG_NUM, A.CDA_TYP_CD,
            D.DM_AV_MODEL_TXT AS CDA_TYP_TXT,
            SUM (CDA_TOT_TRXN_AMT) AS TOT_TRXN_AMT
    FROM     KCDWHPRC.TCDSA A, KCDWHPRC.TDMAV D, SESSION.TT_RPTDT DT
    WHERE    A.CO_ID = DT.CO_ID
    AND      A.CDA_TYP_CD= D.DM_AV_CD
    AND      D.DM_AV_TBL_CD='CDA-TYP-CD'
    AND      CDA_EFF_DT > DT.RPT_DT - 1 YEAR
    AND      CDA_STAT_CD = 'A'
    GROUP BY A.CO_ID, POL_ID, CVG_NUM,A.CDA_TYP_CD, D.DM_AV_MODEL_TXT
);

-- CHECK 8: expected ZERO (matches the INSERT's own GROUP BY)
SELECT CO_ID, POL_ID, CVG_NUM, CDA_TYP_CD, COUNT(*) AS DUP_COUNT
FROM SESSION.TT_CDSA
GROUP BY CO_ID, POL_ID, CVG_NUM, CDA_TYP_CD
HAVING COUNT(*) > 1
ORDER BY DUP_COUNT DESC;


/* ============================================================================
   STEP 9 — TT_LOAN (independent of TT_POL)
============================================================================ */

DROP TABLE SESSION.TT_LOAN;

DECLARE GLOBAL TEMPORARY TABLE SESSION.TT_LOAN AS
(
    SELECT   'INGENIUM' AS SOURCE, LOAN.CO_ID, LOAN.POL_ID,
    VARCHAR (LISTAGG (LOAN.POL_LOAN_ID, ',')WITHIN GROUP (ORDER BY LOAN.POL_LOAN_ID),4) AS LOAN_TYP,
    MAX (CASE WHEN LOAN.POL_LOAN_ID ='C' THEN LOAN.LOAN_AMT ELSE 0 END)AS CASH_LOAN_AMT,
    MAX (CASE WHEN LOAN.POL_LOAN_ID='A' THEN LOAN.LOAN_AMT ELSE 0 END)AS APL_AMT,
    MAX (CASE WHEN LOAN.POL_LOAN_ID ='C' THEN LOAN.LOAN_INT_YTD_AMT ELSE 0 END) AS CASH_LOAN_INT_YTD_AMT,
    MAX (CASE WHEN LOAN.POL_LOAN_ID ='A' THEN LOAN.LOAN_INT_YTD_AMT ELSE 0 END) AS APL_INT_YTD_AMT
    FROM     KCDWHPRC.TPOLL LOAN
    GROUP BY LOAN.CO_ID, LOAN.POL_ID
)
DEFINITION ONLY on commit preserve rows not logged;

INSERT INTO SESSION.TT_LOAN
(
    SELECT   LOAN.SOURCE , LOAN.CO_ID, LOAN.POL_ID,
            CASE WHEN LOAN.POL_ID = APL.POLICY THEN 'APL' ELSE 'CASH' END AS LOAN_TYP_CD,
            LOAN.LOAN_AMT - COALESCE (APL.TOT_APL_AMT,0)AS CASH_LOAN_AMT,
            APL.TOT_APL_AMT AS APL_AMT,
            LOAN.LOAN_INT_BILL_AMT * (LOAN.LOAN_AMT - COALESCE (APL.TOT_APL_AMT,0))/ COALESCE (LOAN.LOAN_AMT,0) AS CASH_LOAN_INT_YTD_AMT,
            LOAN.LOAN_INT_BILL_AMT * APL.TOT_APL_AMT/ COALESCE (LOAN.LOAN_AMT ,0) AS APL_INT_YTD_AMT
    FROM     KCDWHPRC.T8LPOLL LOAN
    INNER JOIN
    (
        SELECT   CO_ID, POL_ID, MAX (EFF_DT)AS MAX_EFF_DT
        FROM     KCDWHPRC.T8LPOLL
        WHERE    CO_ID IN ('01','50')
        AND      POL_ID > '0'
        AND      CRNT_REC_IND = 'Y'
        GROUP BY CO_ID, POL_ID
    ) M
    ON LOAN.CO_ID = M.CO_ID
    AND LOAN.POL_ID = M.POL_ID
    AND LOAN.EFF_DT = M.MAX_EFF_DT
    AND LOAN.LOAN_AMT > 0
    LEFT OUTER JOIN
    (
        SELECT LOAN.CO, LOAN.POLICY, COUNT (LOAN.PDTODT) AS APL_CNT,
        MAX (LOAN.ENTRDT)AS LAST_LOAN_DT, MAX (LOAN.PDTODT) AS LAST_PDTODT,
        SUM (LOAN.APL_AMT)AS TOT_APL_AMT
        FROM
        (
                SELECT CO , POLICY ,   ENTRDT, ACCT.STATUS, ACCT.PDTODT,
                        ACCT_TRXN_CD, SUNDAMT AS APL_AMT
                FROM KCDWHPRC.T8D69 ACCT, SESSION.TT_RPTDT DT
                WHERE CO = DT.CO_ID
                AND CRNT_REC_IND='Y'
                AND ACCT_TRXN_CD ='061'
                AND ACCT.PROGCD = '3'
                AND ACCT.ENTRDT >= DT.RPT_DT - 1 YEAR
        ) LOAN
        INNER JOIN
        (
                SELECT CO , POLICY ,   ENTRDT, ACCT.STATUS, ACCT.PDTODT,
                        ACCT_TRXN_CD, SUNDAMT
                FROM KCDWHPRC.T8D69 ACCT, SESSION.TT_RPTDT DT
                WHERE CO = DT.CO_ID
                AND CRNT_REC_IND='Y'
                AND ACCT_TRXN_CD ='012'
                AND ACCT.PROGCD = '3'
                AND ACCT.ENTRDT >= DT.RPT_DT - 1 YEAR
        ) PREM
        ON LOAN.CO = PREM.CO
        AND LOAN.POLICY = PREM.POLICY
        AND LOAN.ENTRDT = PREM.ENTRDT
        AND LOAN.APL_AMT = PREM.SUNDAMT
        GROUP BY LOAN.CO, LOAN.POLICY
    ) APL
    ON LOAN.CO_ID = APL.CO
    AND LOAN.POL_ID = APL.POLICY
);

-- CHECK 9: expected ZERO
SELECT SOURCE, CO_ID, POL_ID, COUNT(*) AS DUP_COUNT
FROM SESSION.TT_LOAN
GROUP BY SOURCE, CO_ID, POL_ID
HAVING COUNT(*) > 1
ORDER BY DUP_COUNT DESC;


/* ============================================================================
   STEP 10 — TT_CFLW (joins TT_POL, but filtered to SOURCE='INGENIUM' only)
============================================================================ */

DROP TABLE SESSION.TT_CFLW ;

DECLARE GLOBAL TEMPORARY TABLE SESSION.TT_CFLW AS
(
    Select CFLW.CO_ID , CFLW.POL_ID , CF_EFF_DT , CF_SEQ_NUM, PREV_UPDT_TS, CF_FND_BAL_AMT
    from KCDWHPRC.TCFLW CFLW,
    (
        Select C.CO_ID , C.POL_ID , MAX (CF_EFF_DT) AS MAX_CF_EFF_DT , MAX (PREV_UPDT_TS)AS MAX_PREV_UPDT_TS
        from KCDWHPRC.TCFLW C,SESSION.TT_RPTDT DT, SESSION.TT_POL POL
        where C.CO_ID=DT.CO_ID
        AND POL.SOURCE='INGENIUM'
        AND C.CO_ID= POL.CO_ID
        AND C.POL_ID= POL.POL_ID
        AND CF_EFF_DT<= DT.RPT_DT
        AND DATE (PREV_UPDT_TS) <= DT.RPT_DT
        GROUP BY C.CO_ID , C.POL_ID
    )MCFLW
    where CFLW.CO_ID= MCFLW.CO_ID
    AND CFLW.POL_ID= MCFLW.POL_ID
    AND CFLW.CF_EFF_DT= MCFLW.MAX_CF_EFF_DT
    AND CFLW.PREV_UPDT_TS= MCFLW.MAX_PREV_UPDT_TS
)
DEFINITION ONLY on commit preserve rows not logged;

INSERT INTO SESSION.TT_CFLW
(
    Select CFLW.CO_ID , CFLW.POL_ID , CF_EFF_DT , CF_SEQ_NUM, PREV_UPDT_TS, CF_FND_BAL_AMT
    from KCDWHPRC.TCFLW CFLW,
    (
        Select C.CO_ID , C.POL_ID , MAX (CF_EFF_DT) AS MAX_CF_EFF_DT , MAX (PREV_UPDT_TS)AS MAX_PREV_UPDT_TS
        from KCDWHPRC.TCFLW C,SESSION.TT_RPTDT DT, SESSION.TT_POL POL
        where C.CO_ID=DT.CO_ID
        AND POL.SOURCE='INGENIUM'
        AND C.CO_ID= POL.CO_ID
        AND C.POL_ID= POL.POL_ID
        AND CF_EFF_DT<= DT.RPT_DT
        AND DATE (PREV_UPDT_TS) <= DT.RPT_DT
        GROUP BY C.CO_ID , C.POL_ID
    )MCFLW
    where CFLW.CO_ID= MCFLW.CO_ID
    AND CFLW.POL_ID= MCFLW.POL_ID
    AND CFLW.CF_EFF_DT= MCFLW.MAX_CF_EFF_DT
    AND CFLW.PREV_UPDT_TS= MCFLW.MAX_PREV_UPDT_TS
);

-- CHECK 10: expected ZERO
SELECT CO_ID, POL_ID, COUNT(*) AS DUP_COUNT
FROM SESSION.TT_CFLW
GROUP BY CO_ID, POL_ID
HAVING COUNT(*) > 1
ORDER BY DUP_COUNT DESC;


/* ============================================================================
   WHAT WE CANNOT VALIDATE YET

   TT_pe: index creation was captured (Section 15's neighbor), but its
   DECLARE/INSERT logic was NOT - it falls in the missing 978-1269 gap.
   Sections 11-14 (whatever they cover) are also missing entirely.

   MOST IMPORTANT GAP: the actual final SELECT/merge that assembles
   TT_POL + TT_CV + TT_LCV + TT_PUA + TT_POLPUA + TT_DVPCV + TT_CDSA +
   TT_LOAN + TT_CFLW + TT_pe (+ whatever else) into rows for
   INSERT INTO ACTUARIAL.POL_VALU is NOT anywhere in what we've captured.
   Section 16 (the last section in the file) only contains four
   DROP INDEX statements - no SELECT, no INSERT.

   This means we CANNOT yet answer Suresh's central question - does the
   TT_POL duplicate survive into the final table, or does the merge
   logic's join/aggregation naturally collapse it back to one row?

   NEXT STEPS TO CLOSE THIS:
     1. Run everything above and record which CHECK numbers return rows.
     2. Locate the actual final merge/INSERT statement - it may be:
          - In the missing 978-1269 gap of "new 22" itself, or
          - In a different tab entirely (fr300.fex, fr901.fex, and the
            others open alongside "new 22" are good candidates - "new
            22" may be Lokesh's scratch copy of just the session-table
            build portion, with the real final extract living in one of
            the named .fex files).
     3. Once found, trace how it joins CO_ID/POL_ID across all the
        session tables - if it's a plain join without any additional
        SUM/GROUP BY at that final stage, the TT_POL duplicate WILL
        carry through untouched.
============================================================================ */
