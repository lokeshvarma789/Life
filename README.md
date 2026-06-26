
/* ============================================================================
   MEETING PREP - D_MEMBER / D_CLIENT DATA ISSUES
   Meeting with: Mary Beth Daus, Josh Burns, John Steinbeck, Sandra Hotz, Devin Jones
   Reference   : INC0054877, BUG 44624, CHG0039753 (UAT2), CHG0039879 (PROD)
   ----------------------------------------------------------------------------
   Run these queries DURING the meeting (or capture results BEFORE the meeting
   so you can paste them into the discussion).

   ORGANIZED BY DISCUSSION TOPIC:
     PART 1 - D_MEMBER deleted members investigation
     PART 2 - D_CLIENT / D_PLAN / D_COUNCIL duplicates
     PART 3 - Impact analysis (what changes if we apply the filter)
     PART 4 - Cross-view check (other dimensions with same gap)
============================================================================ */

USE ROLE SF_PROD_DATA_ENGINEER;
USE WAREHOUSE PROD_DATA_ENGINEER_WH;
USE DATABASE PROD_DV;


/* ============================================================================
   PART 1 - D_MEMBER DELETED MEMBERS INVESTIGATION
============================================================================ */

-- 1.1 Confirm the current state - how many deleted members in D_MEMBER today
SELECT 
    COUNT(*) AS DELETED_MEMBERS_IN_DMEMBER,
    COUNT(DISTINCT a.MB_NMBR) AS UNIQUE_DELETED_MEMBERS
FROM PROD_DV.INFO_MART.D_MEMBER a
INNER JOIN PROD_DV.BUSINESS_VAULT.S_STD_MEMBER b 
    ON a.MB_NMBR = b.MB_NMBR
WHERE b.MEMBER_DELETE_FLAG = 'Y';


-- 1.2 Show breakdown - how many active vs deleted in D_MEMBER
SELECT 
    b.MEMBER_DELETE_FLAG,
    COUNT(*) AS RECORD_COUNT,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) AS PCT_OF_TOTAL
FROM PROD_DV.INFO_MART.D_MEMBER a
LEFT JOIN PROD_DV.BUSINESS_VAULT.S_STD_MEMBER b 
    ON a.MB_NMBR = b.MB_NMBR
GROUP BY b.MEMBER_DELETE_FLAG
ORDER BY b.MEMBER_DELETE_FLAG NULLS LAST;


-- 1.3 Show a SAMPLE of deleted members so they can see real records
SELECT 
    a.MB_NMBR,
    a.MEMBER_FIRST_NAME,
    a.MEMBER_LAST_NAME,
    a.MEMBER_CLASS_CODE,
    a.MEMBER_EXIT_DATE,
    a.MEMBER_EXIT_TRANSACTION_CODE,
    b.MEMBER_DELETE_FLAG,
    a.LAST_UPDATE_TIMESTAMP
FROM PROD_DV.INFO_MART.D_MEMBER a
INNER JOIN PROD_DV.BUSINESS_VAULT.S_STD_MEMBER b 
    ON a.MB_NMBR = b.MB_NMBR
WHERE b.MEMBER_DELETE_FLAG = 'Y'
LIMIT 10;


/* ============================================================================
   PART 2 - DUPLICATES IN OTHER VIEWS
   Michael said these may not be REAL duplicates - check hash keys
============================================================================ */

-- 2.1 D_CLIENT duplicate count
SELECT 
    'D_CLIENT' AS VIEW_NAME,
    COUNT(*) AS TOTAL_RECORDS,
    COUNT(DISTINCT CLIENT_ID) AS UNIQUE_CLIENTS,
    COUNT(*) - COUNT(DISTINCT CLIENT_ID) AS DUPLICATE_RECORDS
FROM PROD_DV.INFO_MART.D_CLIENT;


-- 2.2 D_CLIENT - check if duplicates have DIFFERENT hash keys (Michael's question)
-- If hash keys differ -> NOT real duplicates (just type-2 history)
-- If hash keys identical -> REAL duplicates needing a fix
SELECT 
    CLIENT_ID,
    COUNT(*) AS RECORD_COUNT,
    COUNT(DISTINCT H_HK_CLIENT) AS DISTINCT_HASH_KEYS,
    CASE 
        WHEN COUNT(*) = COUNT(DISTINCT H_HK_CLIENT) THEN 'NOT_REAL_DUP_TYPE2_HISTORY'
        ELSE 'REAL_DUPLICATE'
    END AS DUPLICATE_TYPE
FROM PROD_DV.INFO_MART.D_CLIENT
GROUP BY CLIENT_ID
HAVING COUNT(*) > 1
ORDER BY RECORD_COUNT DESC
LIMIT 20;


-- 2.3 D_PLAN duplicate analysis
SELECT 
    PLAN_ID,           -- adjust column name if different
    COUNT(*) AS RECORD_COUNT
FROM PROD_DV.INFO_MART.D_PLAN
GROUP BY PLAN_ID
HAVING COUNT(*) > 1
LIMIT 20;


-- 2.4 D_COUNCIL duplicate analysis
SELECT 
    COUNCIL_NUMBER,    -- adjust column name if different
    COUNT(*) AS RECORD_COUNT
FROM PROD_DV.INFO_MART.D_COUNCIL
GROUP BY COUNCIL_NUMBER
HAVING COUNT(*) > 1
LIMIT 20;


-- 2.5 Summary - duplicate count per dimension view
SELECT 'D_MEMBER'  AS VIEW_NAME, COUNT(*) - COUNT(DISTINCT MB_NMBR)         AS DUPS FROM PROD_DV.INFO_MART.D_MEMBER
UNION ALL
SELECT 'D_CLIENT'  AS VIEW_NAME, COUNT(*) - COUNT(DISTINCT CLIENT_ID)        AS DUPS FROM PROD_DV.INFO_MART.D_CLIENT
UNION ALL
SELECT 'D_PLAN'    AS VIEW_NAME, COUNT(*) - COUNT(DISTINCT PLAN_ID)          AS DUPS FROM PROD_DV.INFO_MART.D_PLAN
UNION ALL
SELECT 'D_COUNCIL' AS VIEW_NAME, COUNT(*) - COUNT(DISTINCT COUNCIL_NUMBER)   AS DUPS FROM PROD_DV.INFO_MART.D_COUNCIL;


/* ============================================================================
   PART 3 - IMPACT ANALYSIS (what changes if we apply the filter)
   THIS IS THE KEY ASK - what happens to record counts after the fix
============================================================================ */

-- 3.1 D_MEMBER - before vs after filter impact
WITH 
current_state AS (
    SELECT COUNT(*) AS RECORDS FROM PROD_DV.INFO_MART.D_MEMBER
),
after_fix AS (
    SELECT COUNT(*) AS RECORDS 
    FROM PROD_DV.INFO_MART.D_MEMBER a
    LEFT JOIN PROD_DV.BUSINESS_VAULT.S_STD_MEMBER b ON a.MB_NMBR = b.MB_NMBR
    WHERE b.MEMBER_DELETE_FLAG <> 'Y' OR b.MEMBER_DELETE_FLAG IS NULL
)
SELECT
    c.RECORDS AS CURRENT_DMEMBER_RECORDS,
    a.RECORDS AS AFTER_FILTER_RECORDS,
    c.RECORDS - a.RECORDS AS RECORDS_REMOVED,
    ROUND(100.0 * (c.RECORDS - a.RECORDS) / c.RECORDS, 2) AS PCT_REMOVED
FROM current_state c, after_fix a;


-- 3.2 D_CLIENT - same impact analysis (in case CLIENT_DELETE_FLAG ever populates)
WITH 
current_state AS (
    SELECT COUNT(*) AS RECORDS FROM PROD_DV.INFO_MART.D_CLIENT
),
after_fix AS (
    SELECT COUNT(*) AS RECORDS 
    FROM PROD_DV.INFO_MART.D_CLIENT a
    LEFT JOIN PROD_DV.BUSINESS_VAULT.S_STD_CLIENT b ON a.CLIENT_ID = b.CLIENT_ID
    WHERE b.CLIENT_DELETE_FLAG <> 'Y' OR b.CLIENT_DELETE_FLAG IS NULL
)
SELECT
    c.RECORDS AS CURRENT_DCLIENT_RECORDS,
    a.RECORDS AS AFTER_FILTER_RECORDS,
    c.RECORDS - a.RECORDS AS RECORDS_REMOVED,
    ROUND(100.0 * (c.RECORDS - a.RECORDS) / c.RECORDS, 2) AS PCT_REMOVED
FROM current_state c, after_fix a;


/* ============================================================================
   PART 4 - CROSS-VIEW CHECK
   Same gap may exist in other dimension views - audit them all
============================================================================ */

-- 4.1 Check if every dimension view that has a *_DELETE_FLAG upstream
--     is correctly filtering it. Quick audit per view.

-- D_MEMBER deletes flowing through:
SELECT 'D_MEMBER' AS VIEW_NM, COUNT(*) AS DELETED_LEAKING 
FROM PROD_DV.INFO_MART.D_MEMBER a
INNER JOIN PROD_DV.BUSINESS_VAULT.S_STD_MEMBER b ON a.MB_NMBR = b.MB_NMBR
WHERE b.MEMBER_DELETE_FLAG = 'Y'

UNION ALL

-- D_CLIENT deletes flowing through:
SELECT 'D_CLIENT' AS VIEW_NM, COUNT(*) AS DELETED_LEAKING 
FROM PROD_DV.INFO_MART.D_CLIENT a
INNER JOIN PROD_DV.BUSINESS_VAULT.S_STD_CLIENT b ON a.CLIENT_ID = b.CLIENT_ID
WHERE b.CLIENT_DELETE_FLAG = 'Y';


/* ============================================================================
   PART 5 - SHOW THE VIEW DDL (the root cause)
   Run this to show them that MEMBER_DELETE_FLAG is commented out in the view
============================================================================ */

SELECT GET_DDL('VIEW', 'PROD_DV.INFO_MART.D_MEMBER');

SELECT GET_DDL('VIEW', 'PROD_DV.INFO_MART.D_CLIENT');
