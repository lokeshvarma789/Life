
-*====================================================================================================
-*  Program Name:      af114.fex
-*  Department:        Field Management
-*  Source Tables:     AVRPT_MVT, T9ACB, T9AGC, T9WF2, T9LOC, V_SUL_ISSUE_PREM
-*  Destination Tables: N/A
-*  Number of Reports: 3
-*  Output Format:     EXCEL
-*  Scheduled:         Month End
-*  Frequency:         Monthly
-*  Archive, Purge:    keep 12 months
-*  Purpose:           Report for Field Management/Agency. The report is based on AV017A by state and GA, also by Field Director
-*  Team Track:        31300
-*  Include modules:   KCENVR, KCDATES, KCHD01, KCFT01, KCSTYL01, KCEMPTY, KCKEYWORD
-*  Programs Called:   N/A
-*  Calling Modules:   N/A
-*  Date Created:      05/27/2016
-*  Author:            Emma Schwarz
-*  Report Title:      Life Year to Date Issues
-*  Description:       This report will include information from AV017A.
-*                     The data will be broken down by State and Province and also separately by GA.
-*================ Modification History ===============================================================
-*  Date       Initials Change
-*----------- --------- -------------------------------------------------------------------------------
-* 05-27-2016 ES        TT 31300: Initial revision.
-* 12-14-2016 ES        TT 31300: Updated code to use SSOT tables.
-*                      Removed dependency on AV017A. All queries are included in this procedure.
-* 01-04-2017 MDF       TT 31300: Added mapping of GAs to Regions and a new tab for Regions.
-*                      Until F.M. is confident the Agency table (TZAGY) is up to date, this will be run off-cycle using manual mapping provide by Michele Pannone.
-* 04-17-2017 MDF       TT 31300: Added extract from V_AGENT_INFO/TZAGY/TZACR/T9CLI to pull current GA to Field Director mapping.
-*                      (5/1/2017): TT has been approved. This will use V_AGENT_INFO/TZAGY/TZACR/T9CLI to pull current GA to Field Director mapping.
-*                      Manual method will no longer be used.
-* 03-26-2018 MDF       TT 34530. Use new source for SUL Premium: V_SUL_ISSUE_PREM
-* 02-06-2018 PRI       TT 33913 updated for CSO 2017 Non-LPX. Added 903 and 911 plans grouping.
-*                      Added 911 policies to reduce the Base premium by 90% same as 811 policies.
-* 10-30-2018 PRI       TT 35504: CSO 2017 LPX - Updated 901 Plan grouping based on CVG_PREM_FMT_DUR - # years payable/Paid up Age
-* 01-04-2018 PRI       Add TT 35754 changes to TT 35504: Added CVG_PLAN_ID IN ('842C', '8420', '942C', '9420') for new issues, They were ignored due to the CVG_SUPP_BNFT_CD = 'S' value in AVRPT_MVT
-* 02-04-2019 PRI       TT 35504: CSO 2017 LPX - Added new plan codes for CSO 2017 LPX With additional changes.
-*                                9*P - Income Protection Rider           93PU, 95PU, 96PU, 93PC,95PC, 96PC
-*                                9*PS - IPR with 5% Increase             93PSU, 95PSU, 96PSU, 93P5C, 95P5C, 96P5C
-*                                9SP  - Single Deposit Paid Up Additions Rider   9SPU, 9SPC
-*                                9MAD - Modal Additional Deposit Paid Up Additions Rider 9MADU, 9MADC
-*                                91L - 10 Year LTR - Extended Conversion Period 91LUN, 91LCN
-*                                92C - Child Insurance Rider             92CPUU, 92CU, 92CPUC, 92CC
-*                                92L - 20 Year LTR - Extended Conversion Period 92LUN, 92LCN
-* 03-11-2019 PRI       TT 34445 CSO 2017 LPX - Additional change. Updated the sort order for 901 plans when CVG_PREM_PMT_DUR is 30+.
-* 08-26-2019 PRI       TT 36911 Updated AF114 for UL-G and Added INS_SUB_TYP_CD EQ '2'
-* 12-23-2020 MDF       TT 39129 Ignore old 'Not Taken's
-* 08-11-2021 PRI       TT 40210 Excluded the 4 plans that are used by claims for child riders upon death to base policy, CVG_PLAN_ID IN ('82CPUU', '82CPUC', '92CPUU', '92CPUC')
-*                      Due to emergency promote, 40210 went to production prior to 39816 and 40108 (via Projects)
-* 05-10-2021 MDF       TT 39816 Update product/row mix for 8088 Whole Life Re-price for Nonforfeiture Change
-* 07-07-2021 MDF       TT 40108 Add new rider product: Blended Term Rider (920TBC, 920TBU)
-* 02-14-2022 PRI       TT 40877 Add new rider product: Blended Term Rider 920THC - Canada, 920THU - US
-* 02-06-2025 MDF       RITM0067573 Add new rider products: 901CIC, 901CIU, 911CIC, 911CIU - Chronic Illness Rider
-* 04-03-2026 MDF       RITM0116279 Added new 30YR Term: 878U (30YR Term, US), 878C (30YR Term, Canada). 87730U (Full Conversion Rider, US), 87730C (Full Conversion Rider, Canada)
-* [DATE]     LV        RITM[TICKET#] New Term Portfolio - Added new riders:
-*                      892xx - Accelerator Rider (US: 89210U,89215U,89220U,89230U / Canada: 89210C,89215C,89220C,89230C)
-*                      894xx - Chronic Illness Conversion Rider (US: 89410U,89415U,89420U,89430U / Canada: 89410C,89415C,89420C,89430C)
-*                      890   - Charity Rider (US: 890U / Canada: 890C)
-*                      Added plan codes to CASE WHEN block, ALT_UP_PLAN_DESC_TXT sort block,
-*                      and CVG_SUPP_BNFT_CD carve-out in WHERE clause.
-*                      #NEW_TERM_PORTFOLIO_CHANGES
-*=======================================================================================================

-DEFAULT &WFFMT = 'EXL2K'
-DEFAULT &XRETR = 'ON'
-*
SET ONLINE-FMT  = &WFFMT
SET ASNAMES     = ON
SET HOLDLIST    = PRINTONLY
SET BYDISPLAY   = ON
SET NODATA      = ' '
-*
-SET &FEX_NAME  = 'af114' ;
-SET &RPT_RANGE = 'YM' ;
-*
-*-SET &OVRUSR = 'webfpmo';
-*-SET &OVRDSYSDT = '20260731';
-*
-SET &BASE_TBL = 'AVRPT_MVT';
-*-SET &BASE_TBL = 'AVRPT_MVT_HIST';
-*
-INCLUDE KCENVR
-INCLUDE KCDATES
-*
-SET PERIODICITY = 'Monthly';
-SET &LAST_YEAR  = &END_YEAR - 1;
-*
SET XRETRIEVAL  = &XRETR
-RUN
-*
-SET &VM_WORKFILE = '$Workfile:   af114.fex  $' ;
-SET &VM_REVISION = '$Revision:   1.18  $' ;
-INCLUDE KCKEYWORD
-*
-*Retrieve data
SET XRETRIEVAL=ON
-*
SQL DS2
SELECT   COMPANY,
         POL_ID,
         CVG_NUM,
         BASE_RIDER_IND,
         PLAN_DESC,
         CVG_PLAN_ID,
         POL_ISS_LOC_CD,
         CVG_PREM_PMT_DUR,
         SUM (MVT_TRXN_POL_QTY) AS POLICY_CNT,
         SUM (RIDER_COUNT)      AS RIDER_CNT,
         SUM (MOVEMENT_AMT)     AS SUM_INSURED,
         SUM (CVG_GROSS_PREM)   AS PREMIUM,
         SUM (ENHC_PREM_AMT)    AS ACCEL_TERM
FROM
(
       SELECT   MVT.COMPANY,
                MVT.POL_ID,
                MVT.CVG_NUM,
                MVT.MVT_TRXN_POL_QTY,
                MVT.RIDER_COUNT,
                MVT.MOVEMENT_AMT,
-* Accelerator Term:
                MVT.ENHC_PREM_AMT,
-*34530 update to use Actuarial Premium Calc for SUL
-*TT 36911  Updated NB001 for UL-G and Added INS_SUB_TYP_CD EQ '2'
                CASE WHEN MVT.INS_SUB_TYP_CD IN ('T', '2') THEN VSUL.SUL_PREM_AMT
                                                ELSE MVT.CVG_GROSS_PREM
                END                  AS CVG_GROSS_PREM ,
                MVT.CVG_PLAN_ID,
                MVT.CVG_PREM_PMT_DUR,
                CASE
-* New plan IDs added for  TT 34445

-* TT 39816 Added new plan ids re: Project "8088 Whole Life Re-price for Nonforfeiture Change"
-*          Used existing report line for the new plan IDs.
-*          Only new line is for "911 - Single Premium Whole Life Plan as RRSP".  This previously came out automatically via the ELSE PLAN_GROUP_TXT
-*          condition but now that we're combining 911CR with 911CRB we coded it.

-* TT 40108 Add new rider for Canada, US (Blended Term Rider 920TBC, 920TBU)
-*
-* RITM0067573 - Add new rider products: 901CIC, 901CIU, 911CIC, 911CIU - Chronic Illness Rider

                 WHEN        MVT.CVG_PLAN_ID IN ('93PC','93PU','95PC','95PU','96PC','96PU','93PBC','93PBU','95PBC','95PBU','96PBC','96PBU')         THEN '9*P - Income Protection Rider'
                 WHEN        MVT.CVG_PLAN_ID IN ('93P5C','93P5U','95P5C','95P5U','96P5C','96P5U','93P5BC','93P5BU','95P5BC','95P5BU','96P5BC','96P5BU') THEN '9*PS - IPR with 5% Increase'
                 WHEN        MVT.CVG_PLAN_ID IN ('9SPC' ,'9SPU' ,'913BC' ,'913BU')                                                       THEN '9SP - Single Deposit Paid Up Additions Rider'
                 WHEN        MVT.CVG_PLAN_ID IN ('9MADC','9MADU','914BC' ,'914BU')                                                       THEN '9MAD - Modal Additional Deposit Paid Up Additions Rider'
                 WHEN        MVT.CVG_PLAN_ID IN ('91LCN','91LUN','91LBCN','91LBUN')                                                     THEN '91L - 10 Year LTR - Extended Conversion Period'
                 WHEN        MVT.CVG_PLAN_ID IN ('920TBC','920TBU')                                                                    THEN '920 - Blended Term Rider'
                 WHEN        MVT.CVG_PLAN_ID IN ('920THC','920THU')                                                                    THEN '920 - High PUA Custom Term Blend Rider'
                 WHEN        MVT.CVG_PLAN_ID IN ('92CC' ,'92CU' ,'92CBC' ,'92CBU' ,'92CPBC','92CPBU')                                     THEN '92C - Child Insurance Rider'
                 WHEN        MVT.CVG_PLAN_ID IN ('92LCN','92LUN','92LBCN','92LBUN')                                                     THEN '92L - 20 Year LTR - Extended Conversion Period'
                 WHEN SUBSTR (MVT.CVG_PLAN_ID,1,3) = '801'                                                                                 THEN '801 - Life Paid up at Age 100'
                 WHEN        MVT.CVG_PLAN_ID IN ('803U' ,'803C' ,'803EX')                                                                THEN '803 - Graded Death Benefit Whole Life Plan $5000 FA'
                 WHEN        MVT.CVG_PLAN_ID IN ('803U2','803C2','803EX2')                                                               THEN '803 - Graded Death Benefit Whole Life Plan $7500 FA'
                 WHEN        MVT.CVG_PLAN_ID IN ('901U' ,'901C' ,'901BU' ,'901BC')                                                       THEN '901 - Life Paid up at Age X'
                 WHEN        MVT.CVG_PLAN_ID    IN ('903U' ,'903C' ,'903BC' ,'903BU')                                                    THEN '903 - Graded Death Benefit Whole Life $5,000'
                 WHEN        MVT.CVG_PLAN_ID    IN ('903U2','903C2','903BC2','903BU2')                                                  THEN '903 - Graded Death Benefit Whole Life $10,000'
                 WHEN        MVT.CVG_PLAN_ID    IN ('911U' ,'911C' ,'911BC' ,'911BU')                                                    THEN '911 - Single Premium Whole Life'
                 WHEN        MVT.CVG_PLAN_ID    IN ('911CR','911CRB')                                                                     THEN '911 - Single Premium Whole Life Plan as RRSP'
                 WHEN        MVT.CVG_PLAN_ID    IN ('911CIC','911CIU')                                                                    THEN '911 - Chronic Illness Accelerated Death Benefit Rider'
                 WHEN        MVT.CVG_PLAN_ID    IN ('901CIC','901CIU')                                                                    THEN '901 - Chronic Illness Accelerated Death Benefit Rider'
                 WHEN MVT.INS_SUB_TYP_CD='N' AND MVT.CVG_PLAN_ID IN ( '84PU', '81PU', '82PU', '83PU', '85PU','86PU', '86PC', '83PC', '84PC', '81PC', '85PC', '82PC') THEN '8*P - Income Protection Rider'
                 WHEN MVT.INS_SUB_TYP_CD='J'                                                                                                  THEN '82C - Child Insurance Rider'
                 WHEN MVT.INS_SUB_TYP_CD='R'                                                                                                  THEN '812 - Annual Deposit Paid Up Additions Rider'
                 WHEN MVT.INS_SUB_TYP_CD='S'                                                                                                  THEN '813 - Single Deposit Paid Up Additions Rider'
                 WHEN MVT.INS_SUB_TYP_CD='X'                                                                                                  THEN '814 - Modal Additional Deposit Paid Up Additions Rider'
-* =====================================================================
-* #NEW_TERM_PORTFOLIO_CHANGES - START
-* RITM[TICKET#] New Term Portfolio - Added 3 new riders below
-* Accelerator Rider: 10YR/15YR/20YR/30YR variants for US and Canada
-* Chronic Illness Conversion Rider: 10YR/15YR/20YR/30YR variants for US and Canada
-* Charity Rider (automatic): single plan for US and Canada
-* NOTE: Display label text to be confirmed with business (Field Management / Daniela)
-*       before production promotion. Placeholder labels used below.
-* =====================================================================
                 WHEN        MVT.CVG_PLAN_ID IN ('89210U','89215U','89220U','89230U',
                                                  '89210C','89215C','89220C','89230C')
                                                                                       THEN '892 - Accelerator Rider'
                 WHEN        MVT.CVG_PLAN_ID IN ('89410U','89415U','89420U','89430U',
                                                  '89410C','89415C','89420C','89430C')
                                                                                       THEN '894 - Chronic Illness Conversion Rider'
                 WHEN        MVT.CVG_PLAN_ID IN ('890U','890C')                        THEN '890 - Charity Rider'
-* #NEW_TERM_PORTFOLIO_CHANGES - END
-* =====================================================================
                 ELSE PLAN_GROUP_TXT
           END AS "PLAN_DESC",
           MVT.BASE_RIDER_IND ,
           MVT.POL_ISS_LOC_CD

    FROM &SCHEMA|.&BASE_TBL MVT
-* 34530 update to use Actuarial Premium Calc for SUL
       LEFT OUTER JOIN &SCHEMA|.V_SUL_ISSUE_PREM VSUL
           ON   MVT.COMPANY      = VSUL.COMPANY
           AND  MVT.POL_ID       = VSUL.POL_ID
           AND  MVT.MVT_TRXN_TS  = VSUL.MVT_TRXN_TS
    WHERE    MVT.POL_ID          > '0'
    AND      MVT.MVT_YEAR        = &END_YEAR
    AND      MVT.MVT_DATE        <= '&END_DATE.EVAL'
    AND      MVT.CRNT_REC_IND    = 'Y'
    AND      MVT.TRANSACTION     IN ('Accelerator Term Opt Outs', 'Negative New Issue', 'New Issue', 'New Issue Term Increase')
-*TT 35754: Added CVG_PLAN_ID IN ('842C', '842U', '942C', '942U') for new issues, They were ignored due to the CVG_SUPP_BNFT_CD = 'S' value in AVRPT_MVT
-* =====================================================================
-* #NEW_TERM_PORTFOLIO_CHANGES - WHERE clause carve-out
-* RITM[TICKET#] Added 892xx, 894xx, 890U, 890C to the CVG_SUPP_BNFT_CD exclusion carve-out.
-* Reason: These new rider plan codes may carry CVG_SUPP_BNFT_CD = 'S' which would
-*          cause them to be filtered out without this carve-out — same pattern as
-*          901CIU/901CIC/911CIU/911CIC added in RITM0067573.
-* VERIFY: Confirm CVG_SUPP_BNFT_CD values for 892xx/894xx/890 in DUTDM before
-*         promoting to production. If they are always 'N' or ' ', this carve-out
-*         is harmless but unnecessary. If any are 'S', this carve-out is required.
-* Query to verify: SELECT DISTINCT CVG_PLAN_ID, CVG_SUPP_BNFT_CD
-*                  FROM KCDWHPRC.AVRPT_MVT
-*                  WHERE CVG_PLAN_ID LIKE '892%'
-*                     OR CVG_PLAN_ID LIKE '894%'
-*                     OR CVG_PLAN_ID IN ('890U','890C')
-* #NEW_TERM_PORTFOLIO_CHANGES - WHERE clause
-* =====================================================================
    AND      (MVT.CVG_SUPP_BNFT_CD IN ('N', ' ') OR MVT.CVG_PLAN_ID IN ('842C', '842U', '942C', '942U', '901CIU', '901CIC', '911CIU', '911CIC',
                                                                           '89210U','89215U','89220U','89230U',
                                                                           '89210C','89215C','89220C','89230C',
                                                                           '89410U','89415U','89420U','89430U',
                                                                           '89410C','89415C','89420C','89430C',
                                                                           '890U','890C'))
-*TT 40210: Exclude these 4 plans that are used by claims for child riders upon death to base policy.
    AND      MVT.CVG_PLAN_ID NOT IN ('82CPUU', '82CPUC', '92CPUU', '92CPUC')
-*TT 39129 Ignore old Not Takens
    AND      MVT.POL_ID NOT IN
    (
    SELECT DISTINCT POL_ID
    FROM
    (
    SELECT ROW_NUMBER () OVER
           ( PARTITION BY
                COMPANY,
                POL_ID,
                CVG_NUM,
                MVT_TRXN_TS,
                CRNT_REC_IND
             ORDER BY MVT_TRXN_TS DESC) MVT_RANK,
           COMPANY,
           POL_ID,
           BUS_FCN_DESC_TXT,
           MVT_YEAR,
           ISSUE_YR
    FROM &SCHEMA|.AVRPT_MVT
    WHERE CRNT_REC_IND = 'Y'
    AND MVT_YEAR       = &END_YEAR
    AND ISSUE_YR      < &LAST_YEAR
    )
    --* last transaction should be Not Taken
    WHERE MVT_RANK       = 1
    AND BUS_FCN_DESC_TXT = 'Not Taken'
    )
-*
)
GROUP BY COMPANY, POL_ID, CVG_NUM, BASE_RIDER_IND, PLAN_DESC, CVG_PLAN_ID, POL_ISS_LOC_CD, CVG_PREM_PMT_DUR
ORDER BY COMPANY, POL_ID, CVG_NUM, BASE_RIDER_IND, PLAN_DESC, CVG_PLAN_ID, POL_ISS_LOC_CD, CVG_PREM_PMT_DUR
FOR FETCH ONLY;
TABLE ON TABLE HOLD AS AF114A1
END
-RUN

-*
DEFINE FILE AF114A1
RIDER_FACE_AMT/D17.2CS MISSING ON = IF BASE_RIDER_IND EQ 'RIDER' THEN SUM_INSURED ELSE MISSING ;
POL_FACE_AMT/D17.2CS   MISSING ON = IF BASE_RIDER_IND EQ 'BASE'  THEN SUM_INSURED ELSE MISSING ;
RIDER_PREM/D15.2CS     MISSING ON = IF BASE_RIDER_IND EQ 'RIDER' THEN PREMIUM ELSE MISSING;
BASE_PREM/D15.2CS      MISSING ON = IF BASE_RIDER_IND EQ 'BASE'  THEN PREMIUM ELSE MISSING;
POLICY_TXT/A13 = IF BASE_RIDER_IND EQ 'BASE' THEN 'Base Policies' ELSE 'Riders' ;
-* For sorting
-* TT 35504: CSO 2017 LPX - 901 PLAN SORTING
CVG_901_PMT_DUR/I5 = EDIT (CVG_PREM_PMT_DUR);
-*
-* TT 39816 Added new plan IDs 901BC, 901BU to PLAN_DESCRIPTION AND ALT_UP_PLAN_DESC_TXT
PLAN_DESCRIPTION/A75 = IF CVG_PLAN_ID EQ '901U' OR '901C' OR '901BC' OR '901BU' AND (CVG_901_PMT_DUR GE 5  AND CVG_901_PMT_DUR LE 9)  THEN PLAN_DESC || ': 5-9 Pay'
                       ELSE IF CVG_PLAN_ID EQ '901U' OR '901C' OR '901BC' OR '901BU' AND (CVG_901_PMT_DUR GE 10 AND CVG_901_PMT_DUR LE 14) THEN PLAN_DESC || ': 10-14 Pay'
                       ELSE IF CVG_PLAN_ID EQ '901U' OR '901C' OR '901BC' OR '901BU' AND (CVG_901_PMT_DUR GE 15 AND CVG_901_PMT_DUR LE 19) THEN PLAN_DESC || ': 15-19 Pay'
                       ELSE IF CVG_PLAN_ID EQ '901U' OR '901C' OR '901BC' OR '901BU' AND (CVG_901_PMT_DUR GE 20 AND CVG_901_PMT_DUR LE 29) THEN PLAN_DESC || ': 20-29 Pay'
                       ELSE IF CVG_PLAN_ID EQ '901U' OR '901C' OR '901BC' OR '901BU' AND (CVG_901_PMT_DUR GE 30)                        THEN PLAN_DESC || ': 30+ Pay'
                       ELSE PLAN_DESC;

ALT_UP_PLAN_DESC_TXT/A80 = IF CVG_PLAN_ID EQ '871C' OR '871U' AND PLAN_DESC CONTAINS 'Increase Amount' THEN '874A'
                           ELSE IF CVG_PLAN_ID EQ '872C' OR '872U' AND PLAN_DESC CONTAINS 'Increase Amount' THEN '874B'
                           ELSE IF CVG_PLAN_ID EQ '874C' OR '874U' AND PLAN_DESC CONTAINS 'Increase Amount' THEN '874C'
-* RITM0116279 - A new 30YR Accelerated Term
                           ELSE IF CVG_PLAN_ID EQ '878C' OR '878U' AND PLAN_DESC CONTAINS 'Increase Amount' THEN '874D'
-* =====================================================================
-* #NEW_TERM_PORTFOLIO_CHANGES - ALT_UP_PLAN_DESC_TXT sort block
-* RITM[TICKET#] Added Accelerator Rider 892xx sort entry.
-* The 892 series (Accelerator Rider) follows the same accelerated-term sort
-* pattern as 871/872/874/878. Using '874E' as the next slot in the sequence.
-* NOTE: Chronic Illness Conversion Rider (894xx) and Charity Rider (890U/890C)
-*       are NOT accelerated-term products and do NOT need an entry in this block.
-*       They will fall through to ELSE PLAN_DESCRIPTION which is correct.
-* NOTE: Confirm '874E' sort slot with business / Atif before production promote.
-*       If a different sort position is required, adjust accordingly.
-* #NEW_TERM_PORTFOLIO_CHANGES - ALT_UP_PLAN_DESC_TXT
-* =====================================================================
                           ELSE IF CVG_PLAN_ID EQ '89210C' OR '89210U' OR '89215C' OR '89215U'
                                OR '89220C' OR '89220U' OR '89230C' OR '89230U'
                                AND PLAN_DESC CONTAINS 'Increase Amount'             THEN '874E'
-* =====================================================================
-* #NEW_TERM_PORTFOLIO_CHANGES - END ALT_UP_PLAN_DESC_TXT block
-* =====================================================================
                           ELSE IF CVG_PLAN_ID EQ '901U' OR '901C' OR '901BC' OR '901BU' AND (CVG_901_PMT_DUR GE 5  AND CVG_901_PMT_DUR LE 9)  THEN '901A'
                           ELSE IF CVG_PLAN_ID EQ '901U' OR '901C' OR '901BC' OR '901BU' AND (CVG_901_PMT_DUR GE 10 AND CVG_901_PMT_DUR LE 14) THEN '901B'
                           ELSE IF CVG_PLAN_ID EQ '901U' OR '901C' OR '901BC' OR '901BU' AND (CVG_901_PMT_DUR GE 15 AND CVG_901_PMT_DUR LE 19) THEN '901C'
                           ELSE IF CVG_PLAN_ID EQ '901U' OR '901C' OR '901BC' OR '901BU' AND (CVG_901_PMT_DUR GE 20 AND CVG_901_PMT_DUR LE 29) THEN '901D'
                           ELSE IF CVG_PLAN_ID EQ '901U' OR '901C' OR '901BC' OR '901BU' AND (CVG_901_PMT_DUR GE 30)                        THEN '901E'
                           ELSE PLAN_DESCRIPTION ;

W_CO_ID/A2   WITH COMPANY = '75' ;
W_CO_TXT/A13 WITH COMPANY = 'Whole Society' ;
    CO_TXT/A13 = IF COMPANY EQ '01' THEN 'United States'
                 ELSE IF COMPANY EQ '50' THEN 'Canada' ELSE 'XX' ;
DUM/A1 WITH COMPANY = ' ' ;
END
-*
-* Put all Life data into 1 hold file and eliminate duplicate policies/coverages
-* where a policy has a plus and a minus value.
TABLE FILE AF114A1
SUM RIDER_CNT
    RIDER_FACE_AMT
    POLICY_CNT
    POL_FACE_AMT
    BASE_PREM
    RIDER_PREM
    BASE_RIDER_IND
BY POL_ISS_LOC_CD
BY DUM
BY COMPANY
BY CO_TXT
BY POL_ID
BY CVG_NUM
BY POLICY_TXT
BY PLAN_DESCRIPTION
BY ALT_UP_PLAN_DESC_TXT
ON TABLE HOLD AS AF114AR
END
-RUN

-*
-* Get GA id and name
-GA_ID
SQL DB2
SELECT DISTINCT T9ACB.CO_ID
     , T9ACB.POL_ID
     , T9ACB.CVG_NUM
     , T9AGC.GA_AGT_ID
, RTRIM(SUBSTR(T9WF2.AGT_INDV_GIV_NM,1,10)) || ' ' ||SUBSTR(T9WF2.AGT_INDV_MID_NM,1,1) || ' ' ||RTRIM(SUBSTR(T9WF2.AGT_INDV_SUR_NM,1,15)) GANAME
FROM &SCHEMA|.T9ACB T9ACB
JOIN &SCHEMA|.T9AGC T9AGC
     ON T9AGC.AGT_CNTRCT_ID = T9ACB.AGT_CNTRCT_ID
     AND T9AGC.CRNT_REC_IND = 'Y'
JOIN &SCHEMA|.T9WF2 T9WF2
     ON T9AGC.GA_AGT_ID = T9WF2.AGT_ID
     AND T9WF2.CRNT_REC_IND = 'Y'
WHERE T9ACB.CRNT_REC_IND = 'Y'
ORDER BY T9ACB.POL_ID, T9ACB.CVG_NUM
FOR FETCH ONLY;
TABLE ON TABLE HOLD AS AF114XGA
END
-RUN
-* Eliminate any duplicate records for a policy-coverage combination
TABLE FILE AF114XGA
SUM FST.GA_AGT_ID
    FST.GANAME
BY POL_ID
BY CVG_NUM
ON TABLE HOLD AS AF114GA
END
-RUN

-*
-* Get location description:
SQL DB2
SELECT DISTINCT T9LOC.LOC_CD
     , T9LOC.LOC_DESC_TXT
FROM &SCHEMA|.T9LOC T9LOC
WHERE T9LOC.CRNT_REC_IND = 'Y'
  AND T9LOC.CTRY_2_POSN_CD IN ('CA','US')
ORDER BY T9LOC.LOC_CD
FOR FETCH ONLY;
TABLE ON TABLE HOLD AS AF114LOC
END
-RUN

-*
SET ALL=OFF
JOIN CLEAR *
JOIN POL_ISS_LOC_CD IN AF114AR TO LOC_CD IN AF114LOC AS J1
END
-*
-* Create data file including location. Sort data for join to GA info.
TABLE FILE AF114AR
SUM RIDER_CNT
    RIDER_FACE_AMT
    POLICY_CNT
    POL_FACE_AMT
    BASE_PREM
    RIDER_PREM
    PLAN_DESCRIPTION AS PLAN_DESC_TXT
    LOC_DESC_TXT
BY POL_ID
BY CVG_NUM
BY DUM
BY COMPANY
BY CO_TXT
BY POLICY_TXT
BY ALT_UP_PLAN_DESC_TXT
ON TABLE HOLD AS AF114AG
END
-RUN

-*
-*
-*Get Field Directors
-*
SQL DB2
SELECT DISTINCT
A.GA_AGT_ID,
C.CLI_INDV_SUR_NM
FROM
        &SCHEMA|.V_AGENT_INFO A,
        &SCHEMA|.TZACR        B,
        &SCHEMA|.T9CLI        C,
        &SCHEMA|.TZAGY        D
WHERE
        D.AGNCY_ID           = B.AGNCY_ID
AND B.AGT_ID                = A.AGT_ID
AND D.AGNCY_BR_FD_CLI_ID    = C.CLI_ID
AND C.CRNT_REC_IND          = 'Y'
AND A.AGT_CNTRCT_TYP_CD     = 'A'
AND A.AGT_STAT_CD           = 'A'
AND A.AGT_ID                = A.GA_AGT_ID
AND A.AGT_CNTRCT_EFF_DT =   (SELECT MAX(A2.AGT_CNTRCT_EFF_DT)
                             FROM &SCHEMA|.T9AGC A2
                             WHERE A2.AGT_CNTRCT_EFF_DT <= '&END_DATE'
                               AND A2.GA_CNTRCT_EFF_DT  <= '&END_DATE'
                               AND A2.CO_ID             = A.CO_ID
                               AND A2.AGT_ID            = A.AGT_ID)
AND A.AGC_EFF_DT        =   (SELECT MAX(A3.EFF_DT)
                             FROM &SCHEMA|.T9AGC A3
                             WHERE A3.AGT_CNTRCT_EFF_DT <= '&END_DATE'
                               AND A3.GA_CNTRCT_EFF_DT  <= '&END_DATE'
                               AND A3.EFF_DT            <= '&END_DATE'
                               AND A3.CO_ID             = A.CO_ID
                               AND A3.AGT_ID            = A.AGT_ID
                               AND A3.GA_CNTRCT_EFF_DT  = A.GA_CNTRCT_EFF_DT
                               AND A3.AGT_CNTRCT_EFF_DT = A.AGT_CNTRCT_EFF_DT)
AND A.GA_CNTRCT_EFF_DT  =   (SELECT MAX(A4.GA_CNTRCT_EFF_DT)
                             FROM &SCHEMA|.T9AGC A4
                             WHERE A4.AGT_CNTRCT_EFF_DT <= '&END_DATE'
                               AND A4.GA_CNTRCT_EFF_DT  <= '&END_DATE'
                               AND A4.CO_ID             = A.CO_ID
                               AND A4.AGT_ID            = A.AGT_ID
                               AND A4.AGT_CNTRCT_EFF_DT = A.AGT_CNTRCT_EFF_DT)
AND     (A.AGT_CNTRCT_END_DT IS NULL OR A.AGT_CNTRCT_END_DT >= '&END_DATE')
AND     (A.GA_CNTRCT_END_DT  IS NULL OR A.GA_CNTRCT_END_DT  >= '&END_DATE')
AND B.AGT_CNTRCT_END_DT IS NULL
AND B.AGT_ID >= '002000'
ORDER BY A.GA_AGT_ID
FOR FETCH ONLY;
-*
TABLE ON TABLE HOLD AS AF114FD FORMAT FOCUS INDEX GA_AGT_ID
END
-RUN
-*


-* Join to merge policy and GA info
SET ALL=OFF
JOIN CLEAR *
JOIN POL_ID AND CVG_NUM IN AF114AG TO POL_ID AND CVG_NUM IN AF114GA AS J2
JOIN GA_AGT_ID          IN AF114AG TO GA_AGT_ID          IN AF114FD AS J3
END
-*
-* Starting Life YTD reports
-* Create total data for US and Canada in correct format for reports
DEFINE FILE AF114AG
T_RIDER_CNT/D8CS        MISSING ON = RIDER_CNT ;
T_RIDER_FACE_AMT/D17CS  MISSING ON = RIDER_FACE_AMT ;
T_POLICY_CNT/D8CS       MISSING ON = POLICY_CNT ;
T_POL_FACE_AMT/D17CS    MISSING ON = POL_FACE_AMT ;
T_BASE_PREM/D17CS       MISSING ON = BASE_PREM ;
T_RIDER_PREM/D17CS      MISSING ON = RIDER_PREM ;
BLANK/A1 WITH COMPANY = ' ' ;
CO_TOT_TXT/A23 = 'Total for ' | CO_TXT ;
POL_TOT_TXT/A23 = 'Total for ' | POLICY_TXT ;
STATE_TOT_TXT/A90 = 'Total for ' | LOC_DESC_TXT ;
-* For Whole Society report
WS_TOT_TXT/A13 = IF COMPANY EQ '01' THEN 'Totals US'
                 ELSE IF COMPANY EQ '50' THEN 'Totals Canada' ELSE 'YYYY' ;
REGION_NAME/A25 = IF CLI_INDV_SUR_NM EQ '' THEN 'ZZ-Other' ELSE CLI_INDV_SUR_NM;
-* For Region report
REG_SUB_TXT/A38 = 'Region: '|REGION_NAME;
REG_TOT_TXT/A48 = 'Total for ' | REG_SUB_TXT ;

-* For additional totals across policy/rider:
-*Total Count - this will be the same as total policy count as Joe does not want to include riders in this count
-*Use T_POLICY_CNT
-*Total Sum Insured - this will be the total of Rider Sum Insured plus Policy Sum Insured
T_SUM_INS/D17CS = T_RIDER_FACE_AMT + T_POL_FACE_AMT ;
-*Total Premium - this will be the total of Rider Premium plus Base premium with this exception:
-*Single Premium plans will be reduced by 90% in this total. So actual value will be displayed in
-*rider or base premium but only 10% of that value for 811, 812, 813 and 814 plans will be included when calculating this total.
-* Get first 3 characters of the plan desc text containing the plan:
PLAN3/A3 = SUBSTR(80,PLAN_DESC_TXT,1,3,3,PLAN3);
-* Reduce base prem by 90% for 811/911 plans
-*BASE_PREM_T/D17CS = IF PLAN3 EQ '811' THEN T_BASE_PREM/90 ELSE T_BASE_PREM ;
BASE_PREM_T/D17CS = IF PLAN3 EQ '811' OR '911' THEN (T_BASE_PREM * .10) ELSE T_BASE_PREM ;

-* no changes to riders for CSO TT 33913 note
-* Reduce rider prem by 90% for 812, 813 and 814 plans
-*RIDR_PREM_T/D17CS = IF PLAN3 EQ '812' OR '813' OR '814' THEN (T_RIDER_PREM/90) ELSE T_RIDER_PREM ;
RIDR_PREM_T/D17CS = IF PLAN3 EQ '812' OR '813' OR '814' THEN (T_RIDER_PREM * .10) ELSE T_RIDER_PREM ;
-* Calculate total premium (prior to reduction in 811-814 plans)
ALL_PREM/D17CS     = T_BASE_PREM + T_RIDER_PREM;
-* Calculate new total premium (after reduction in 811-814 and 911 plans)
T_PREM/D17CS = BASE_PREM_T + RIDR_PREM_T ;
END
-RUN
-*
-SET &RPT_TITLE   = 'Life Year to Date Issues Summary by State/Province' ;
-* Create the life ytd reports. Loop created so code is not repeated.
-* Controlling variable:
-SET &AF114YTD = 0 ;
-AF114A_LYTD
-SET &AF114YTD = &AF114YTD + 1 ;
-SET &LIFE_RPT = IF &AF114YTD EQ 1 THEN 'US'
                 ELSE IF &AF114YTD EQ 2 THEN 'Canada'
                 ELSE '' ;
-SET &RPT_TITLE2 = '<CO_TXT';
-SET &TITLETEXT = IF &AF114YTD EQ 1 THEN 'Life YTD Issues US'
                  ELSE IF &AF114YTD EQ 2 THEN 'Life YTD Issues Canada'
                  ELSE '' ;
-SET &CNTRY_ORDER = IF &AF114YTD EQ 1 THEN '01'
                    ELSE IF &AF114YTD EQ 2 THEN '50' ELSE '75' ;
-SET &COMPOUND = IF &AF114YTD EQ 1 THEN 'OPEN' ELSE ' ' ;
SET COMPOUND     = &COMPOUND
-RUN
-SKIP_COMP
-*
-*
TABLE FILE AF114AG
HEADING CENTER
-INCLUDE KCHD01
"" ""
SUM PLAN_DESC_TXT       AS ''
    T_RIDER_CNT         AS ''
    T_RIDER_FACE_AMT    AS ''
    T_RIDER_PREM        AS ''
    T_POLICY_CNT        AS ''
    T_POL_FACE_AMT      AS ''
    T_BASE_PREM         AS ''
    CO_TOT_TXT          NOPRINT
    POL_TOT_TXT         NOPRINT
    STATE_TOT_TXT       NOPRINT
    BLANK               NOPRINT
    T_SUM_INS           NOPRINT
    T_PREM              NOPRINT
    ALL_PREM            NOPRINT

COMPUTE CNT_DESC/I5 = IF LOC_DESC_TXT      EQ LAST LOC_DESC_TXT
                      AND POLICY_TXT       EQ LAST POLICY_TXT
                      AND ALT_UP_PLAN_DESC_TXT NE LAST ALT_UP_PLAN_DESC_TXT
                      AND LAST CNT_DESC LE 4 THEN CNT_DESC + 1
                      ELSE 1 ; NOPRINT
-* Insert WHERE based on which page is being generated:
WHERE COMPANY EQ '&CNTRY_ORDER.EVAL'
BY DUM             NOPRINT REPAGE
BY COMPANY         AS ''
BY LOC_DESC_TXT    NOPRINT SUBHEAD
"<0> <LOC_DESC_TXT"
"<0> <0> <0>Rider<0>Rider<0>Rider<0>Policy<0>Policy<0>Base<0>Total<0>Total<0>Total<0>Total"
"<0>Plan Description<0>Count**<0>Sum Insured<0>Premium<0>Count<0>Sum Insured<0>Premium<0>Count<0>Sum Insured<0>Premium<0>Premium****"
ON LOC_DESC_TXT    SUBFOOT
"<ST.BLANK<ST.STATE_TOT_TXT<ST.T_RIDER_CNT<ST.T_RIDER_FACE_AMT<ST.T_RIDER_PREM<ST.T_POLICY_CNT<ST.T_POL_FACE_AMT<ST.T_BASE_PREM<ST.T_POLICY_CNT<ST.T_SUM_INS<ST.ALL_PREM<ST.T_PREM "
"" ""
"Asterisk (*) denotes combining of all IPR plan codes. (10 year, 15 year, 20 year, 25 year, 30 year, and to age 65)"
" (**) Counts are informational only, not included in totals."
" (***) Total reflects 10% for single premium plans 811, 911, 812, 813 and 814."
"" ""
BY POLICY_TXT      NOPRINT
ON POLICY_TXT      SUBFOOT
"<ST.BLANK<ST.POL_TOT_TXT<ST.T_RIDER_CNT<ST.T_RIDER_FACE_AMT<ST.T_RIDER_PREM<ST.T_POLICY_CNT<ST.T_POL_FACE_AMT<ST.T_BASE_PREM "
"" ""
BY ALT_UP_PLAN_DESC_TXT NOPRINT
ON TABLE NOTOTAL
FOOTING BOTTOM
-INCLUDE KCFT01
"" ""
"Notes: Does not include Not-Taken and declined policies."
"       The Policy Premium amount represents the Premium Paying and On Waiver policies only."
ON TABLE SET STYLESHEET *
-INCLUDE KCSTYL01
-*
TYPE=HEADING, HEADALIGN=BODY, $
TYPE=HEADING, LINE=1, COLSPAN=12,$
TYPE=HEADING, LINE=2, COLSPAN=12,$
TYPE=HEADING, LINE=3, COLSPAN=12,$
TYPE=HEADING, LINE=4, ITEM=1, COLSPAN=12,$
TYPE=HEADING, LINE=5, COLSPAN=12,SIZE=8,$
-*
TYPE=SUBHEAD, HEADALIGN=BODY, $
TYPE=SUBHEAD, BY=LOC_DESC_TXT, LINE=1, ITEM=2, JUSTIFY=CENTER, $
TYPE=SUBHEAD, BY=LOC_DESC_TXT, LINE=2, JUSTIFY=CENTER, BACKCOLOR=NAVY,COLOR=WHITE,$
TYPE=SUBHEAD, BY=LOC_DESC_TXT, LINE=3, JUSTIFY=CENTER, BACKCOLOR=NAVY,COLOR=WHITE,$
-*
TYPE=DATA, BACKCOLOR=RGB(236 236 236), WHEN=CNT_DESC EQ 5,$
-*
TYPE=TABFOOTING, COLSPAN=5, SIZE=9,$
TYPE=REPORT, TITLETEXT='&TITLETEXT.EVAL', $
TYPE=REPORT, COLUMN=PLAN_DESC_TXT, WRAP=3.6, $
TYPE=REPORT, COLUMN=PLAN1_DESC_TXT, WRAP=3.6, $
TYPE=REPORT, COLUMN=T_RIDER_CNT,     WRAP=.6,  JUSTIFY=RIGHT,$
TYPE=REPORT, COLUMN=T_RIDER_FACE_AMT,WRAP=1,   JUSTIFY=RIGHT,$
TYPE=REPORT, COLUMN=T_RIDER_PREM,    WRAP=1,   JUSTIFY=RIGHT,$
TYPE=REPORT, COLUMN=T_POLICY_CNT,    WRAP=.55, JUSTIFY=RIGHT,$
TYPE=REPORT, COLUMN=T_POL_FACE_AMT,  WRAP=1,   JUSTIFY=RIGHT,$
TYPE=REPORT, COLUMN=T_BASE_PREM,     WRAP=1,   JUSTIFY=RIGHT,$
TYPE=REPORT, COLUMN=ALL_PREM,        WRAP=1,   JUSTIFY=RIGHT,$
TYPE=SUBTOTAL, STYLE=BOLD, BACKCOLOR=SILVER,$
TYPE=SUBFOOT, HEADALIGN=BODY, $
TYPE=SUBFOOT, BY=POLICY_TXT, LINE=1, ITEM=1, COLSPAN=1, $
TYPE=SUBFOOT, BY=POLICY_TXT, LINE=1, ITEM=2, COLSPAN=1, $
TYPE=SUBFOOT, BY=POLICY_TXT, LINE=1, ITEM=3, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=SUBFOOT, BY=POLICY_TXT, LINE=1, ITEM=4, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=SUBFOOT, BY=POLICY_TXT, LINE=1, ITEM=5, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=SUBFOOT, BY=POLICY_TXT, LINE=1, ITEM=6, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=SUBFOOT, BY=POLICY_TXT, LINE=1, ITEM=7, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=SUBFOOT, BY=POLICY_TXT, LINE=1, ITEM=8, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=SUBFOOT, BY=LOC_DESC_TXT, LINE=1, ITEM=1, COLSPAN=1, $
TYPE=SUBFOOT, BY=LOC_DESC_TXT, LINE=1, ITEM=2, COLSPAN=1, $
TYPE=SUBFOOT, BY=LOC_DESC_TXT, LINE=1, ITEM=3, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=SUBFOOT, BY=LOC_DESC_TXT, LINE=1, ITEM=4, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=SUBFOOT, BY=LOC_DESC_TXT, LINE=1, ITEM=5, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=SUBFOOT, BY=LOC_DESC_TXT, LINE=1, ITEM=6, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=SUBFOOT, BY=LOC_DESC_TXT, LINE=1, ITEM=7, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=SUBFOOT, BY=LOC_DESC_TXT, LINE=1, ITEM=8, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=SUBFOOT, BY=LOC_DESC_TXT, LINE=1, ITEM=9, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=SUBFOOT, BY=LOC_DESC_TXT, LINE=1, ITEM=10, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=SUBFOOT, BY=LOC_DESC_TXT, LINE=1, ITEM=11, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=SUBFOOT, BY=LOC_DESC_TXT, LINE=1, ITEM=12, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=SUBFOOT, BY=LOC_DESC_TXT, LINE=2, ITEM=1, COLSPAN = 12, $
TYPE=SUBFOOT, BY=LOC_DESC_TXT, LINE=3, ITEM=1, COLSPAN = 12, $
TYPE=SUBFOOT, BY=LOC_DESC_TXT, LINE=4, ITEM=1, COLSPAN = 12, $
TYPE=SUBFOOT, BY=LOC_DESC_TXT, LINE=5, ITEM=1, COLSPAN = 12, $
TYPE=SUBFOOT, BY=LOC_DESC_TXT, LINE=6, ITEM=1, COLSPAN = 12, $
TYPE=SUBFOOT, BY=LOC_DESC_TXT, LINE=7, ITEM=1, COLSPAN = 12, $
TYPE=FOOTING, HEADALIGN=INTERNAL, $
TYPE=FOOTING, LINE=2, ITEM=1, COLSPAN = 12, $
TYPE=FOOTING, LINE=3, ITEM=1, COLSPAN = 12, JUSTIFY = RIGHT, $
TYPE=FOOTING, LINE=4, ITEM=1, COLSPAN = 12, SIZE=9, STYLE=BOLD, $
TYPE=FOOTING, LINE=5, ITEM=1, COLSPAN = 12, SIZE=9, STYLE=BOLD, $
TYPE=FOOTING, LINE=6, ITEM=1, COLSPAN = 12, SIZE=9, STYLE=BOLD, $
ENDSTYLE
END
-RUN
-SET &RPT_TITLE2 = ' ' ;
-INCLUDE KCEMPTY
-RUN
-IF &AF114YTD GE 2 GOTO END_AF114A ;
-GOTO AF114A_LYTD
-END_AF114A
-RUN
-*
-*
-* Whole Society summary report
-SET &RPT_TITLE2 = 'Whole Society';
-SET &TITLETEXT  = 'Life YTD-Summary Whole Society' ;
-SET &COMPOUND = ' ' ;
SET COMPOUND = &COMPOUND
TABLE FILE AF114AG
HEADING CENTER
-INCLUDE KCHD01
"" ""
SUM T_RIDER_CNT       AS ''
    T_RIDER_FACE_AMT  AS ''
    T_RIDER_PREM      AS ''
    T_POLICY_CNT      AS ''
    T_POL_FACE_AMT    AS ''
    T_BASE_PREM       AS ''
    T_POLICY_CNT      AS ''
    T_SUM_INS         AS ''
    T_PREM            AS ''
    ALL_PREM          AS ''
    BLANK             NOPRINT
BY DUM                AS '' REPAGE
BY COMPANY            NOPRINT
BY WS_TOT_TXT         AS ''
ON DUM                SUBFOOT
"<ST.BLANK<0>Totals Whole Society<ST.T_RIDER_CNT<ST.T_RIDER_FACE_AMT<ST.T_RIDER_PREM<ST.T_POLICY_CNT<ST.T_POL_FACE_AMT<ST.T_BASE_PREM<ST.T_POLICY_CNT<ST.T_SUM_INS<ST.ALL_PREM<ST.T_PREM "
"" ""
" (**) Counts are informational only, not included in totals."
" (***) Total reflects 10% for single premium plans 811, 911, 812, 813 and 814."
ON DUM                SUBHEAD
"<0> <0> <0>Rider<0>Rider<0>Rider<0>Policy<0>Policy<0>Base<0>Total<0>Total<0>Total<0>Total "
"<0> <0> <0>Count**<0>Sum Insured<0>Premium<0>Count<0>Sum Insured<0>Premium<0>Count<0>Sum Insured<0>Premium<0>Premium*** "
FOOTING BOTTOM
-INCLUDE KCFT01
ON TABLE SET STYLESHEET *
-INCLUDE KCSTYL01
TYPE=HEADING, HEADALIGN=BODY, $
TYPE=HEADING, LINE=1, COLSPAN=12,$
TYPE=HEADING, LINE=2, COLSPAN=12,$
TYPE=HEADING, LINE=3, COLSPAN=12,$
TYPE=HEADING, LINE=4, ITEM=1, COLSPAN=12,$
TYPE=HEADING, LINE=5, COLSPAN=12,SIZE=8,$
-*
TYPE=SUBHEAD, HEADALIGN=BODY, $
TYPE=SUBHEAD, BY=DUM, LINE=1, JUSTIFY=CENTER, BACKCOLOR=NAVY,COLOR=WHITE,$
TYPE=SUBHEAD, BY=DUM, LINE=2, JUSTIFY=CENTER, BACKCOLOR=NAVY,COLOR=WHITE,$
-*
TYPE=TABFOOTING, COLSPAN=5, SIZE=9,$
TYPE=REPORT, TITLETEXT='&TITLETEXT.EVAL', $
TYPE=REPORT, COLUMN=DUM, WRAP=.2, $
TYPE=REPORT, COLUMN=PLAN_DESC_TXT, WRAP=3.6, $
TYPE=REPORT, COLUMN=PLAN1_DESC_TXT, WRAP=3.6, $
TYPE=REPORT, COLUMN=T_RIDER_CNT,     WRAP=.6,  JUSTIFY=RIGHT,$
TYPE=REPORT, COLUMN=T_RIDER_FACE_AMT,WRAP=1,   JUSTIFY=RIGHT,$
TYPE=REPORT, COLUMN=T_RIDER_PREM,    WRAP=1,   JUSTIFY=RIGHT,$
TYPE=REPORT, COLUMN=T_POLICY_CNT,    WRAP=.55, JUSTIFY=RIGHT,$
TYPE=REPORT, COLUMN=T_POL_FACE_AMT,  WRAP=1,   JUSTIFY=RIGHT,$
TYPE=REPORT, COLUMN=T_BASE_PREM,     WRAP=1,   JUSTIFY=RIGHT,$
TYPE=REPORT, COLUMN=T_SUM_INS,       WRAP=1,   JUSTIFY=RIGHT,$
TYPE=REPORT, COLUMN=T_PREM,          WRAP=1,   JUSTIFY=RIGHT,$
TYPE=REPORT, COLUMN=ALL_PREM,        WRAP=1,   JUSTIFY=RIGHT,$
-*
TYPE=SUBFOOT, HEADALIGN=BODY, $
TYPE=SUBFOOT, BY=DUM, LINE=1, ITEM=1, COLSPAN=1, $
TYPE=SUBFOOT, BY=DUM, LINE=1, ITEM=2, COLSPAN=1, $
TYPE=SUBFOOT, BY=DUM, LINE=1, ITEM=3, JUSTIFY=RIGHT, $
TYPE=SUBFOOT, BY=DUM, LINE=1, ITEM=4, JUSTIFY=RIGHT, $
TYPE=SUBFOOT, BY=DUM, LINE=1, ITEM=5, JUSTIFY=RIGHT, $
TYPE=SUBFOOT, BY=DUM, LINE=1, ITEM=6, JUSTIFY=RIGHT, $
TYPE=SUBFOOT, BY=DUM, LINE=1, ITEM=7, JUSTIFY=RIGHT, $
TYPE=SUBFOOT, BY=DUM, LINE=1, ITEM=8, JUSTIFY=RIGHT, $
TYPE=SUBFOOT, BY=DUM, LINE=1, ITEM=9, JUSTIFY=RIGHT, $
TYPE=SUBFOOT, BY=DUM, LINE=1, ITEM=10,JUSTIFY=RIGHT, $
TYPE=SUBFOOT, BY=DUM, LINE=1, ITEM=11,JUSTIFY=RIGHT, $
TYPE=SUBFOOT, BY=DUM, LINE=1, ITEM=12,JUSTIFY=RIGHT, $
-*
TYPE=SUBFOOT, BY=DUM, LINE=2, ITEM=1, COLSPAN = 12, $
TYPE=SUBFOOT, BY=DUM, LINE=3, ITEM=1, COLSPAN = 12, $
TYPE=SUBFOOT, BY=DUM, LINE=4, ITEM=1, COLSPAN = 12, $
ENDSTYLE
END
-RUN
-SET &RPT_TITLE2 = ' ' ;
-INCLUDE KCEMPTY
-RUN
-*
-* Region report
-SET &RPT_TITLE   = 'Life Year to Date Issues by Region' ;
-SET &RPT_TITLE2  = ' ' ;
-SET &TITLETEXT   = 'Life YTD-by Region'

-SET &COMPOUND = 'CLOSE' ;
-*
TABLE FILE AF114AG
HEADING CENTER
-INCLUDE KCHD01
"" ""
SUM COMPANY             AS ''
    PLAN_DESC_TXT       AS ''
    T_RIDER_CNT         AS ''
    T_RIDER_FACE_AMT    AS ''
    T_RIDER_PREM        AS ''
    T_POLICY_CNT        AS ''
    T_POL_FACE_AMT      AS ''
    T_BASE_PREM         AS ''
    CO_TOT_TXT          NOPRINT
    POL_TOT_TXT         NOPRINT
    STATE_TOT_TXT       NOPRINT
    BLANK               NOPRINT
    T_SUM_INS           NOPRINT
    T_PREM              NOPRINT
    ALL_PREM            NOPRINT
COMPUTE CNT_DESC/I5 = IF REGION          EQ LAST REGION
                      AND POLICY_TXT       EQ LAST POLICY_TXT
                      AND ALT_UP_PLAN_DESC_TXT NE LAST ALT_UP_PLAN_DESC_TXT
                      AND LAST CNT_DESC LE 4 THEN CNT_DESC + 1
                      ELSE 1 ; NOPRINT
BY REGION_NAME NOPRINT SUBHEAD
"<0> <REG_SUB_TXT"
"<0> <0> <0>Rider<0>Rider<0>Rider<0>Policy<0>Policy<0>Base<0>Total<0>Total<0>Total<0>Total"
"<0>Plan Description<0>Count**<0>Sum Insured<0>Premium<0>Count<0>Sum Insured<0>Premium<0>Count<0>Sum Insured<0>Premium<0>Premium****"
ON REGION_NAME  SUBFOOT
"<ST.BLANK<ST.REG_TOT_TXT<ST.T_RIDER_CNT<ST.T_RIDER_FACE_AMT<ST.T_RIDER_PREM<ST.T_POLICY_CNT<ST.T_POL_FACE_AMT<ST.T_BASE_PREM<ST.T_POLICY_CNT<ST.T_SUM_INS<ST.ALL_PREM<ST.T_PREM "
"" ""
"Asterisk (*) denotes combining of all IPR plan codes. (10 year, 15 year, 20 year, 25 year, 30 year, and to age 65)"
" (**) Counts are informational only, not included in totals."
" (***) Total reflects 10% for single premium plans 811, 911, 812, 813 and 814."
"" ""
BY POLICY_TXT      NOPRINT
ON POLICY_TXT      SUBFOOT
"<ST.BLANK<ST.POL_TOT_TXT<ST.T_RIDER_CNT<ST.T_RIDER_FACE_AMT<ST.T_RIDER_PREM<ST.T_POLICY_CNT<ST.T_POL_FACE_AMT<ST.T_BASE_PREM "
"" ""
BY ALT_UP_PLAN_DESC_TXT NOPRINT
ON TABLE NOTOTAL
FOOTING BOTTOM
-INCLUDE KCFT01
"" ""
"Notes: Does not include Not-Taken and declined policies."
"       The Policy Premium amount represents the Premium Paying and On Waiver policies only."
ON TABLE SET STYLESHEET *
-INCLUDE KCSTYL01
TYPE=HEADING, HEADALIGN=BODY, $
TYPE=HEADING, LINE=1, COLSPAN=12,$
TYPE=HEADING, LINE=2, COLSPAN=12,$
TYPE=HEADING, LINE=3, COLSPAN=12,$
TYPE=HEADING, LINE=4, ITEM=1, COLSPAN=12,$
TYPE=HEADING, LINE=5, COLSPAN=12,SIZE=8,$
-*
TYPE=SUBHEAD, HEADALIGN=BODY, $
TYPE=SUBHEAD, BY=REGION_NAME, LINE=1, ITEM=2, JUSTIFY=CENTER, $
TYPE=SUBHEAD, BY=REGION_NAME, LINE=2, JUSTIFY=CENTER, BACKCOLOR=NAVY,COLOR=WHITE,$
TYPE=SUBHEAD, BY=REGION_NAME, LINE=3, JUSTIFY=CENTER, BACKCOLOR=NAVY,COLOR=WHITE,$
-*
TYPE=DATA, BACKCOLOR=RGB(236 236 236), WHEN=CNT_DESC EQ 5,$
-*
TYPE=TABFOOTING, COLSPAN=5, SIZE=9,$
TYPE=REPORT, TITLETEXT='&TITLETEXT.EVAL', $
TYPE=REPORT, COLUMN=PLAN_DESC_TXT, WRAP=3.6, $
TYPE=REPORT, COLUMN=PLAN1_DESC_TXT, WRAP=3.6, $
TYPE=REPORT, COLUMN=T_RIDER_CNT,     WRAP=.6,  JUSTIFY=RIGHT,$
TYPE=REPORT, COLUMN=T_RIDER_FACE_AMT,WRAP=1,   JUSTIFY=RIGHT,$
TYPE=REPORT, COLUMN=T_RIDER_PREM,    WRAP=1,   JUSTIFY=RIGHT,$
TYPE=REPORT, COLUMN=T_BASE_PREM,     WRAP=1,   JUSTIFY=RIGHT,$
TYPE=SUBTOTAL, STYLE=BOLD, BACKCOLOR=SILVER,$
TYPE=SUBFOOT, HEADALIGN=BODY, $
-*
TYPE=SUBFOOT, BY=REGION_NAME,  LINE=1, ITEM=1, COLSPAN=1, $
TYPE=SUBFOOT, BY=REGION_NAME,  LINE=1, ITEM=2, COLSPAN=1, $
TYPE=SUBFOOT, BY=REGION_NAME,  LINE=1, ITEM=3, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=SUBFOOT, BY=REGION_NAME,  LINE=1, ITEM=4, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=SUBFOOT, BY=REGION_NAME,  LINE=1, ITEM=5, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=SUBFOOT, BY=REGION_NAME,  LINE=1, ITEM=6, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=SUBFOOT, BY=REGION_NAME,  LINE=1, ITEM=7, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=SUBFOOT, BY=REGION_NAME,  LINE=1, ITEM=8, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=SUBFOOT, BY=REGION_NAME,  LINE=1, ITEM=9, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=SUBFOOT, BY=REGION_NAME,  LINE=1, ITEM=10, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=SUBFOOT, BY=REGION_NAME,  LINE=1, ITEM=11, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=SUBFOOT, BY=REGION_NAME,  LINE=1, ITEM=12, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=SUBFOOT, BY=REGION_NAME,  LINE=2, ITEM=1, COLSPAN = 12, $
TYPE=SUBFOOT, BY=REGION_NAME,  LINE=3, ITEM=1, COLSPAN = 12, $
TYPE=SUBFOOT, BY=REGION_NAME,  LINE=4, ITEM=1, COLSPAN = 12, $
TYPE=SUBFOOT, BY=REGION_NAME,  LINE=5, ITEM=1, COLSPAN = 12, $
-*
TYPE=SUBFOOT, BY=POLICY_TXT, LINE=1, ITEM=1, COLSPAN=1, $
TYPE=SUBFOOT, BY=POLICY_TXT, LINE=1, ITEM=2, COLSPAN=1, $
TYPE=SUBFOOT, BY=POLICY_TXT, LINE=1, ITEM=3, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=SUBFOOT, BY=POLICY_TXT, LINE=1, ITEM=4, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=SUBFOOT, BY=POLICY_TXT, LINE=1, ITEM=5, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=SUBFOOT, BY=POLICY_TXT, LINE=1, ITEM=6, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=SUBFOOT, BY=POLICY_TXT, LINE=1, ITEM=7, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=SUBFOOT, BY=POLICY_TXT, LINE=1, ITEM=8, JUSTIFY=RIGHT, BACKCOLOR=RGB(220 220 220), $
TYPE=FOOTING, HEADALIGN=INTERNAL, $
TYPE=FOOTING, LINE=2, ITEM=1, COLSPAN = 12, $
TYPE=FOOTING, LINE=3, ITEM=1, COLSPAN = 12, JUSTIFY = RIGHT, $
TYPE=FOOTING, LINE=4, ITEM=1, COLSPAN = 12, SIZE=9, STYLE=BOLD, $
TYPE=FOOTING, LINE=5, ITEM=1, COLSPAN = 12, SIZE=9, STYLE=BOLD, $
TYPE=FOOTING, LINE=6, ITEM=1, COLSPAN = 12, SIZE=9, STYLE=BOLD, $
ENDSTYLE
ON TABLE PCHOLD FORMAT EXL2K BYTOC
END
-RUN
-SET &RPT_TITLE2 = ' ' ;
-INCLUDE KCEMPTY
