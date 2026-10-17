-- =====================================================================
-- AV111 - Eligibility section (LTC summary) rebuilt in SQL from the .fex
-- DB2 LUW.  Reproduces the "Eligibility:" block of Summary_US / Summary_CAN
-- for one plan type, as of a chosen end date.
--
-- .fex logic reproduced:
--   ELIG     : blank ELIG_CD -> 'W' if GENDER_CD = 'F', else 'I'; otherwise ELIG_CD
--   STATUS   : one status per policy, first match wins (same order as the
--              DEFINE STATUS/A3 chain in the .fex)
--   Cancelled / Lapsed : counted straight from the status code (not STATUS),
--              so a policy issued and cancelled in the same year is counted
--              in both Issued and Cancelled (as the .fex does)
--   In-Force : status CH/IS/RN/WP, no date filter
--
-- Change the parameters in the P row only.
-- =====================================================================

WITH P (START_DT, END_DT, CO_ID, PLAN_CD) AS (
    VALUES (DATE('2026-01-01'),   -- start of year
            DATE('2026-08-31'),   -- report end date (use 2026-09-30 for the Sept rerun)
            '01',                 -- '01' = US, '50' = Canada
            '2')                  -- '2' = LTC PLUS, '0' = KOC CARE, '9' = total of both
),

BASE AS (
    SELECT
        -- Eligibility code, same as DEFINE ELIG/A2
        CASE
            WHEN COALESCE(TRIM(A.ELIG_CD), '') = '' AND A.GENDER_CD = 'F' THEN 'W'
            WHEN COALESCE(TRIM(A.ELIG_CD), '') = ''                       THEN 'I'
            ELSE TRIM(A.ELIG_CD)
        END AS ELIG,

        A.LNG_TRM_CARE_STAT_CD AS STAT_CD,
        A.STAT_DT,
        A.ISS_DT,

        -- One status per policy, same precedence as DEFINE STATUS/A3
        CASE
            WHEN A.LNG_TRM_CARE_STAT_CD IN ('CA','CB','CC','CE','CF','CH','IS','LA','RE','RN','WP')
                 AND A.ISS_DT  BETWEEN P.START_DT AND P.END_DT THEN 'IS'   -- Issued
            WHEN A.LNG_TRM_CARE_STAT_CD = 'DE'
                 AND A.STAT_DT BETWEEN P.START_DT AND P.END_DT THEN 'DE'   -- Declined
            WHEN A.LNG_TRM_CARE_STAT_CD = 'PP'
                 AND A.STAT_DT BETWEEN P.START_DT AND P.END_DT THEN 'PP'   -- Postponed
            WHEN A.LNG_TRM_CARE_STAT_CD = 'WD'
                 AND A.STAT_DT BETWEEN P.START_DT AND P.END_DT THEN 'WD'   -- Withdrawn
            WHEN A.LNG_TRM_CARE_STAT_CD = 'CF'
                 AND A.STAT_DT BETWEEN P.START_DT AND P.END_DT THEN 'BE'   -- Benefits exhausted
            WHEN A.LNG_TRM_CARE_STAT_CD IN ('CA','CD','RE')
                 AND A.STAT_DT BETWEEN P.START_DT AND P.END_DT THEN 'CA'   -- Cancelled
            WHEN A.LNG_TRM_CARE_STAT_CD IN ('LA','CB','CC')
                 AND A.STAT_DT BETWEEN P.START_DT AND P.END_DT THEN 'LA'   -- Lapsed
            WHEN A.LNG_TRM_CARE_STAT_CD = 'WP'                 THEN 'PW'   -- On claim (prems waived)
            WHEN A.LNG_TRM_CARE_STAT_CD IN ('AP','PN')         THEN 'PE'   -- Pending/approved
            WHEN A.LNG_TRM_CARE_STAT_CD IN ('CH','IS','RN','WP') THEN 'IN' -- In force
            ELSE 'NA'
        END AS STATUS,

        P.START_DT, P.END_DT
    FROM   KCDWHPRC.T8LT1 A
    CROSS JOIN P
    WHERE  A.CRNT_REC_IND = 'Y'
      AND  A.CO_ID = P.CO_ID
      AND (P.PLAN_CD = '9' OR A.LTC_PLAN_TYP_CD = P.PLAN_CD)
      -- AND A.POL_IND <> 'WS'   -- uncomment if the US summary excludes DC priests (see note)
)

SELECT
    CASE WHEN GROUPING(ELIG) = 1 THEN 'TOTAL (must equal YTD row)'
         ELSE CASE ELIG
                WHEN 'I' THEN 'Insurance Member (I)'
                WHEN 'A' THEN 'Associate Member (A)'
                WHEN 'V' THEN 'Inactive Insurance Member (V)'
                WHEN 'W' THEN 'Wife (W)'
                WHEN 'P' THEN 'Prior Member (P)'
                WHEN 'Q' THEN 'Spouse of Prior Member (Q)'
                WHEN 'N' THEN 'Non-Member (N)'
                WHEN 'F' THEN 'Spouse of Non-Member (F)'
                WHEN 'S' THEN 'Spouse''s Right (S)'
                ELSE 'NOT ON REPORT (' || ELIG || ')'
              END
    END                                                              AS ELIGIBILITY,

    SUM(CASE WHEN STATUS = 'IS' THEN 1 ELSE 0 END)                    AS ISSUED,
    SUM(CASE WHEN STATUS = 'DE' THEN 1 ELSE 0 END)                    AS DECLINED,
    SUM(CASE WHEN STATUS = 'PP' THEN 1 ELSE 0 END)                    AS POSTPONED,
    SUM(CASE WHEN STATUS = 'WD' THEN 1 ELSE 0 END)                    AS WITHDRAWN,
    SUM(CASE WHEN STAT_CD IN ('CA','CD','RE')
              AND STAT_DT BETWEEN START_DT AND END_DT THEN 1 ELSE 0 END) AS CANCELLED,
    SUM(CASE WHEN STAT_CD IN ('LA','CB','CC')
              AND STAT_DT BETWEEN START_DT AND END_DT THEN 1 ELSE 0 END) AS LAPSED,
    SUM(CASE WHEN STATUS = 'BE' THEN 1 ELSE 0 END)                    AS BENEFITS_EXHAUSTED,
    SUM(CASE WHEN STATUS = 'PW' THEN 1 ELSE 0 END)                    AS ON_CLAIM_PREMS_WAIVED,
    SUM(CASE WHEN STATUS = 'PE' THEN 1 ELSE 0 END)                    AS PENDING_APPROVED,
    SUM(CASE WHEN STAT_CD IN ('CH','IS','RN','WP') THEN 1 ELSE 0 END) AS IN_FORCE
FROM   BASE
GROUP BY ROLLUP (ELIG)
ORDER BY GROUPING(ELIG),
         CASE ELIG WHEN 'I' THEN 1 WHEN 'A' THEN 2 WHEN 'V' THEN 3 WHEN 'W' THEN 4
                   WHEN 'P' THEN 5 WHEN 'Q' THEN 6 WHEN 'N' THEN 7 WHEN 'F' THEN 8
                   WHEN 'S' THEN 9 ELSE 99 END;
