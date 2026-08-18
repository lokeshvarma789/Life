
KNIGHTS OF COLUMBUS
Data Modernization — Production Support
SNOWFLAKE CT TABLE
VALIDATION & MONITORING RUNBOOK
Guide procedure, parameterized SQL, and escalation workflow
Document Control	
Version	4.0 — SQL Reference Update
Source workspace	Snowflake_Validation_queries (Snowflake Workspaces)
Folders documented	Drill down queries (Guide, Step 2/4/5/6/8/9) · Monitoring queries
Prepared by	Lokesh Varma
Reviewed by	Siddesh Gannu
Status	Updated with full Guide text and production SQL for Steps 2, 4, 5, 6, 8
Open item	Step 9.sql body, Monitoring query.sql, and Validation query.sql not yet captured — see Appendix C
 
Table of Contents
Table of Contents	1
1. Purpose & Scope	1
1.1 Two Ingestion Paths, One Validation Pattern	1
2. Workspace Reference	1
2.1 Drill down queries	1
2.2 Monitoring queries	1
3. Daily Monitoring Procedure (the Guide)	1
Step 1 — Qlik Monitoring: Check Health of Replications	1
Step 2 — Review CT Tables in Snowflake (ING, FRATDB, RPLUS)	1
Step 3 — Verify Talend Production Jobs	1
Step 4 — Review CT Tables in L70, DCLM, SEI, SF, SAP, LTC	1
Step 5 — Validate CT Loads Against Historical Trends	1
Step 6 — Identify Outlier Tables by Insert/Update/Delete Counts	1
Sort and Shortlist Tables	1
Compare Against Historical Trends	1
Flag Potential Outliers	1
Make a Note of Your Findings	1
Step 7 — Reconcile Discrepancies Against Source (AQT)	1
Confirm with Source System	1
Compare Record Counts	1
Perform Detailed Validation (if counts don't match)	1
Make a Note of Your Findings	1
Step 8 — Review CT Tables Not Loaded Today	1
Identify Missing Loads	1
Validate Expected Behavior	1
Investigate Unexpected Gaps	1
Make a Note of Your Findings	1
Step 9 — Review FMC Schema / Airflow DAG Loads (Raw Vault)	1
Validate DAG Execution Status	1
Validate Data Load and Performance	1
Compare Against Historical Trends	1
Flag and Investigate Anomalies	1
Make a Note of Your Findings	1
Steps 10–12 — Escalate & Confirm	1
Procedure	1
Step 13 — Develop the Fix (Dev Environment)	1
Steps 14–19 — Promote the Fix to Production	1
1. Create a Service Request for deployment to Test	1
2. Create an RFC for UAT → UAT2 Deployment	1
3. UAT2 Testing	1
4. Promote to Prod	1
Appendix A — SQL Script Reference	1
A.1 — Step 2.sql: Qlik CT Table Replication Validation (Q-QLIK)	1
A.2 — Step 4.sql: Talend CT Table Load Validation	1
A.3 — Step 5.sql: Single-Table Deep-Dive Anomaly Detection	1
A.4 — Step 6.sql: Schema-Wide Anomaly Ranking	1
A.5 — Step 8.sql: Missing Table Load Detection & Freshness	1
Appendix B — Escalation Quick Reference	1
Appendix C — Open Items for the Next Update	1

 
1. Purpose & Scope
This runbook is the single reference for daily Snowflake CT (Change Table) monitoring across the Data Modernization pipeline. It combines two things that used to live separately: Siddesh's step-by-step monitoring procedure (the "Guide"), and the actual parameterized SQL that implements each step, as built out in the Snowflake_Validation_queries workspace.
CT tables are the backbone of the entire pipeline — every downstream layer (Raw Vault, Business Vault, Info Mart, Power BI) depends on CT tables loading correctly and on schedule. The goal of this monitoring is to catch a broken or stalled load before the business notices it in a report.
Scope of this update:  This version adds the full text of the Guide (Steps 1–19) and the complete SQL from Step 2.sql, Step 4.sql, Step 5.sql, Step 6.sql, and Step 8.sql, transcribed directly from the workspace. Step 9.sql, Monitoring query.sql, and Validation query.sql were referenced but their bodies were not captured in this pass — see Appendix C.
1.1 Two Ingestion Paths, One Validation Pattern
Path	Tool	Schemas covered	Cadence quirks
Database sources	Qlik Cloud (CDC replication)	ING, FRATDB, RPLUS, FRAT_ING	None — expected to refresh continuously
File / batch sources	Talend (TMC-scheduled jobs)	L70, DCLM, SAP, DI, LTC, SF, REF (SEI pending)	L70 has built-in Sunday and Monday gap exceptions (see Step 4)

2. Workspace Reference
All monitoring queries live in the Snowflake_Validation_queries project under Workspaces. The project is split into two folders:
2.1 Drill down queries
File	Purpose
Guide	The full step-by-step monitoring procedure (Steps 1–19). Documented in Section 3.
Step 2.sql	Qlik CT table replication validation (Q-QLIK). Documented in Appendix A.1.
Step 4.sql	Talend CT table load validation. Documented in Appendix A.2.
Step 5.sql	Single-table deep-dive anomaly detection (3-method statistical check). Documented in Appendix A.3.
Step 6.sql	Schema-wide version of the Step 5 logic — ranks every CT table in a schema. Documented in Appendix A.4.
Step 8.sql	Missing table load detection & data freshness monitoring for a single table. Documented in Appendix A.5.
Step 9.sql	FMC / Airflow DAG monitoring in the Raw Vault. Referenced in Guide Step 9 — code not yet captured, see Appendix C.

2.2 Monitoring queries
File	Purpose
FMC_Airflow_Dag.sql	Supports Guide Step 9 — not yet captured in this document, see Appendix C.
Monitoring query.sql	Per-table record count query referenced during daily monitoring — not yet captured, see Appendix C.
Validation query.sql	General validation query, referenced alongside Monitoring query.sql — not yet captured, see Appendix C.
 
3. Daily Monitoring Procedure (the Guide)
This is the operational sequence exactly as maintained in the Guide file. Follow the steps in order — each step tells you which SQL script to run and what a healthy result looks like.
Step 1 — Qlik Monitoring: Check Health of Replications
As part of Qlik monitoring, verify that the Ingenium, ResultsPlus, and FratDB replications are running successfully and loading data into the target tables without issues. Follow the defined Qlik monitoring steps.
Step 2 — Review CT Tables in Snowflake (ING, FRATDB, RPLUS)
Check the CT tables in the ING, FRATDB, and RPLUS Snowflake schemas to confirm data was loaded successfully. Use the HEADER__TIMESTAMP column to verify the latest load date and ensure it aligns with the replication run time.
Run:  Step 2.sql — see Appendix A.1 for the full script.
Step 3 — Verify Talend Production Jobs
Review all Talend jobs in Prod to ensure they are running successfully, following the defined Talend monitoring process.
Step 4 — Review CT Tables in L70, DCLM, SEI, SF, SAP, LTC
Check the CT tables in the L70, DCLM, SEI, SF, SAP, and LTC Snowflake schemas to confirm Talend data loads were successful. Use the HEADER__TIMESTAMP column to verify the latest load date aligns with the job run time in TMC.
Run:  Step 4.sql — see Appendix A.2 for the full script.
Step 5 — Validate CT Loads Against Historical Trends
Validate CT table loads from Qlik and Talend by checking HEADER__TIMESTAMP and ensuring the number of records inserted, updated, or deleted today aligns with historical trends (no anomalies or outliers).
•	Using HEADER__TIMESTAMP, identify all CT tables loaded today from both Qlik and Talend.
•	For each identified table, validate that records were loaded for the current day.
•	Verify today's load volume aligns with historical trends by comparing to previous runs.
•	Assess whether today's load volume falls within the expected range using statistical measures (e.g. standard deviation).
•	Flag the load if it deviates by roughly 2–3 standard deviations from historical norms for further investigation.
Essentially, confirm there is no unusual spike or drop in the data load, and that load volume and frequency remain consistent with typical behavior.
Run:  Step 5.sql (single-table deep dive — edit the target table before running) — see Appendix A.3.
Step 6 — Identify Outlier Tables by Insert/Update/Delete Counts
Identify CT tables loaded today by Qlik or Talend that have the highest and lowest counts of inserted, updated, and deleted records, and compare them against historical trends to flag potential outliers.
Sort and Shortlist Tables
•	Sort CT tables loaded by Qlik or Talend by the number of records inserted, updated, or deleted today (ascending order).
•	Focus on tables with the lowest and highest number of records inserted/updated/deleted today, as both can indicate potential issues. Investigate these further.
Compare Against Historical Trends
•	Analyze historical volumes of records inserted, updated, or deleted per day for each table to understand typical behavior.
•	Establish expected ranges or patterns based on past load trends.
•	Compare today's insert/update/delete counts against these expected ranges to identify deviations.
•	Use statistical measures such as standard deviation to determine whether today's load deviates significantly (e.g. by 2–3 standard deviations) from historical norms; if so, investigate the underlying cause.
Flag Potential Outliers
•	Highlight tables where today's insert/update/delete counts significantly deviate from historical norms.
Example: If a table typically has 200–300 records inserted/updated/deleted daily but only shows ~100 today, flag it for review.
•	Use statistical measures such as interquartile range (IQR) to define upper and lower bounds for outlier detection, and standard deviation to assess whether today's load deviates significantly (e.g. 2–3 standard deviations) from historical norms; if it does, investigate the underlying cause.
Make a Note of Your Findings
Note these for follow-up, with the intent to either document them in the "Observations/Anomalies" section of the Snowflake Issues Tracker or raise them as an incident, as appropriate. Escalate your findings to Siddesh immediately to confirm if an Incident needs to be created.
Run:  Step 6.sql (schema-wide — set target_schema before running) — see Appendix A.4.
Step 7 — Reconcile Discrepancies Against Source (AQT)
If any discrepancies are identified in Step 6, validate the data load by querying the source system using AQT and comparing record counts with the corresponding target tables to ensure data consistency.
Confirm with Source System
•	Use AQT (Advanced Query Tool) to directly query the source system tables.
•	Validate that the data present in the source aligns with what is expected for the load to the target.
Compare Record Counts
•	Run record count queries on both source tables (via AQT) and target/loaded tables (snapshot tables).
•	Ensure counts match or fall within acceptable thresholds.
Perform Detailed Validation (if counts don't match)
•	Identify missing, duplicate, or mismatched records.
Make a Note of Your Findings
•	Capture any differences, including record count variances and data inconsistencies.
•	Note whether the issue originates from the source, transformation logic, or load process.
•	Escalate your findings to Siddesh immediately to confirm if an Incident needs to be created.
Step 8 — Review CT Tables Not Loaded Today
Review all CT tables that were not loaded today to determine whether this is expected behavior; for any unexpected gaps, investigate the root cause to understand why those tables were not loaded.
Identify Missing Loads
•	Generate a list of CT tables with no records inserted, updated, or deleted today.
Validate Expected Behavior
•	Confirm whether these tables are not expected to load daily (e.g. based on schedule, load frequency, dependencies, or business logic). Use the validation query to check load frequency and schedule.
Investigate Unexpected Gaps
•	For tables that should have been loaded, investigate the root cause, including potential upstream data issues, Qlik replication or Talend job failures, as well as pipeline dependencies or configuration problems.
Make a Note of Your Findings
•	Note these for follow-up; document in "Observations/Anomalies" or raise as an incident. Escalate to Siddesh immediately.
Run:  Step 8.sql (single table — edit the target table before running) — see Appendix A.5.
Step 9 — Review FMC Schema / Airflow DAG Loads (Raw Vault)
Review the FMC schema in the Raw Vault to monitor Airflow DAG loads from Prod Landing, focusing on ING, FRAT_ING, and FRATDB DAGs, to ensure successful runs and that record counts align with historical trends.
Validate DAG Execution Status
•	Focus on Airflow DAGs for ING, FRAT_ING, and FRATDB (typically prefixed with "FMC_3D").
•	Confirm that all scheduled DAGs have completed successfully and that data has been loaded as expected.
•	Identify any failed or long-running DAGs with the help of the FMC schema.
Validate Data Load and Performance
•	Check the number of records processed by each DAG and verify it matches the volumes normally observed for that load.
•	Confirm data is flowing correctly from Prod Landing to the Raw Vault.
•	Identify long-running DAGs, particularly those exceeding typical runtimes (e.g. ~2–3 standard deviations above normal).
Compare Against Historical Trends
•	Analyze historical load volumes for each DAG.
•	Compare today's load volumes against expected volumes derived from historical trends.
•	Compare the current runtime to historical runtimes to ensure it falls within the normal expected range for the DAG.
Flag and Investigate Anomalies
•	Identify potential anomalies in records loaded, DAG runtimes, and DAG execution status (e.g. failures or missed runs) using the FMC schema in the Raw Vault.
•	Investigate potential causes such as ingestion delays, pipeline failures, upstream data issues, or unexpected changes in source data.
Make a Note of Your Findings
•	Make note of any anomalies or irregularities for later review to determine whether they should be logged as observations/anomalies or escalated as an incident. Escalate your findings to Siddesh immediately.
Run:  Step 9.sql — not yet captured in this document. See Appendix C.
Steps 10–12 — Escalate & Confirm
Escalate any identified or unusual issues immediately and confirm with members of the team whether the behavior is expected. Update the Snowflake Issues Tracker and raise an incident accordingly.
Procedure
1.	Consolidate the findings you made a note of in Steps 6, 7, 8, and 9.
2.	If any anomalies, inconsistencies, or potential issues are identified, escalate the findings to your manager (Siddesh) with all relevant details.
3.	Reach out to one of the following team members to validate whether the observed behavior is expected:
–	Joshua Burns
–	Maribeth Daus
–	John Steinbeck
–	Devin Jones
4.	Based on the feedback received:
–	If the behavior is confirmed as expected — document the observations in the "Observations/Anomalies" sheet.
–	If the behavior is confirmed as an issue — create an incident following the standard incident management process (KofC – Incident Management Process v5.docx) and record the details in the "Snowflake Prod Incidents" sheet.
Step 13 — Develop the Fix (Dev Environment)
Develop and implement the fix in the Dev environment. Thoroughly test it to confirm it is functioning as expected.
Steps 14–19 — Promote the Fix to Production
Once the fix is thoroughly tested and confirmed to be working as expected, follow the steps below. Keep Siddesh informed throughout the entire process.
1. Create a Service Request for deployment to Test
•	Contact Peter to create an ADO ticket for testers.
•	Peter will assign the ticket to Lionel or Sam for testing in the Test and UAT environments.
–	Send the ticket number to the Daily Modernization – Daily Build and Bug Scrum group chat and tag Peter Pirro. Include a brief one-line description of the task and what it resolves.
–	Create the Service Request for the promote to Test first. Once the fix is deployed and tested, raise the Service Request to promote it from Test → UAT. Repeat the tag/description step above. Once the fix is promoted to UAT, coordinate with the tester to test the fix.
2. Create an RFC for UAT → UAT2 Deployment
•	After successful completion of the previous step, create an RFC (Change Request) for deployment from UAT to UAT2, following the same tag/description process as above.
3. UAT2 Testing
•	Peter will assign Sam or Lionel to perform testing in the UAT2 environment. After the promote to UAT2, coordinate with the tester to test the fix.
4. Promote to Prod
•	If testing in UAT2 is successful, create an RFC (Change Request) to promote the changes from UAT2 to Prod, following the same process used for the UAT → UAT2 promotion — the only difference is that testing will be performed in Prod.
 
Appendix A — SQL Script Reference
Full scripts, transcribed as-built from the workspace. Each script is a stored, reusable query in the Drill down queries folder — several are parameterized and must be edited (schema, table name) before running against a new target.
A.1 — Step 2.sql: Qlik CT Table Replication Validation (Q-QLIK)
Field	Detail
Purpose	Confirm Qlik-loaded CT tables show a recent HEADER__TIMESTAMP, indicating CDC replication is active.
Scanned schemas	ING, FRATDB, RPLUS, FRAT_ING (only FRATDB is currently un-commented / active)
Source database	DEV_LANDING (per USE DATABASE at top of script)
FRESH	Last HEADER__TIMESTAMP within 4 hours
STALE_TODAY	Updated today, but 4–24 hours old
MISSED_TODAY	No updates today

USE ROLE SF_DEV_DATA_ENGINEER;
USE WAREHOUSE DEV_DATA_ENGINEER_WH;
USE DATABASE DEV_LANDING;
 
-- ============================================================
-- Q-QLIK
-- Qlik CT Table Replication Validation
-- (Step 2 in Siddesh's checklist)
-- ============================================================
 
-- PURPOSE:
--   Confirm Qlik-loaded CT tables show recent HEADER__TIMESTAMP
--   indicating CDC replication is active.
--
-- SCANNED SCHEMAS:
--   ING, FRATDB, RPLUS, FRAT_ING
--
-- VALIDATION RULES:
--   FRESH          -> within 4 hours
--   STALE_TODAY    -> updated today but 4-24 hours old
--   MISSED_TODAY   -> no updates today
--
-- ============================================================
 
EXECUTE IMMEDIATE $$
DECLARE
    sql_stmt STRING;
    rs RESULTSET;
BEGIN
 
    SELECT LISTAGG(
        'SELECT ''' || TABLE_SCHEMA || ''' AS SCHEMA_NAME, ' ||
        '''' || TABLE_NAME || ''' AS TABLE_NAME, ' ||
        '''QLIK CLOUD'' AS TOOL, ' ||
        'MAX(HEADER__TIMESTAMP) AS LAST_HEADER_TS, ' ||
        'DATEDIFF(HOUR, MAX(HEADER__TIMESTAMP), SYSDATE()) AS HOURS_SINCE_LAST, ' ||
        'DATEDIFF(DAY, MAX(HEADER__TIMESTAMP), CURRENT_DATE()) AS DAYS_STALE, ' ||
        'CASE ' ||
        '  WHEN DATEDIFF(HOUR, MAX(HEADER__TIMESTAMP), SYSDATE()) <= 4 THEN ''FRESH - CDC active'' ' ||
        '  WHEN TO_DATE(MAX(HEADER__TIMESTAMP)) = CURRENT_DATE() THEN ''STALE_TODAY - check Qlik task'' ' ||
        '  ELSE ''MISSED_TODAY - investigate replication'' ' ||
        'END AS VALIDATION_STATUS ' ||
        'FROM DEV_LANDING.' || TABLE_SCHEMA || '.' || TABLE_NAME || ' ' ||
        'HAVING MAX(HEADER__TIMESTAMP) IS NOT NULL',
        ' UNION ALL '
    )
    INTO :sql_stmt
    FROM DEV_LANDING.INFORMATION_SCHEMA.TABLES
    WHERE TABLE_TYPE = 'BASE TABLE'
      AND TABLE_SCHEMA IN ('FRATDB')
      --'ING','FRATDB','RPLUS','FRAT_ING'
      AND TABLE_NAME LIKE '%__CT';
 
    sql_stmt := sql_stmt || '
        ORDER BY DAYS_STALE DESC,
                 HOURS_SINCE_LAST DESC';
 
    rs := (EXECUTE IMMEDIATE :sql_stmt);
    RETURN TABLE(rs);
 
END;
$$;
To activate the other schemas:  Uncomment the 'ING','FRATDB','RPLUS','FRAT_ING' line and remove the single-schema 'FRATDB' filter above it.
A.2 — Step 4.sql: Talend CT Table Load Validation
Field	Detail
Purpose	Confirm Talend-loaded CT tables show a recent HEADER__TIMESTAMP, indicating the scheduled load completed.
Scanned schemas	L70, DCLM, SAP, DI, LTC, SF, REF (SEI currently commented out)
Source database	PROD_LANDING
L70 Sunday exception	If today is Sunday and L70's last load is ≤1 day old, flagged EXPECTED_GAP instead of MISSED
L70 Monday exception	If today is Monday and L70's last load is ≤2 days old, flagged EXPECTED_GAP instead of MISSED
Standard logic	LOADED_TODAY / LOADED_YESTERDAY (check TMC if daily table) / MISSED — investigate Talend TMC job

-- ... (see top of file: USE ROLE / WAREHOUSE / DATABASE, header comment block)
--   Empty tables excluded
--   Must confirm actual job in Talend TMC
-- ============================================================
 
EXECUTE IMMEDIATE $$
DECLARE
    sql_stmt STRING;
    rs RESULTSET;
BEGIN
 
    SELECT LISTAGG(
        'SELECT ''' || TABLE_SCHEMA || ''' AS SCHEMA_NAME, ' ||
        '''' || TABLE_NAME || ''' AS TABLE_NAME, ' ||
        '''TALEND'' AS TOOL, ' ||
        'MAX(HEADER__TIMESTAMP) AS LAST_HEADER_TS, ' ||
        'DATEDIFF(DAY, TO_DATE(MAX(HEADER__TIMESTAMP)), CURRENT_DATE()) AS DAYS_SINCE_LAST, ' ||
        'CASE ' ||
 
        -- L70 Sunday exception
        '  WHEN ''' || TABLE_SCHEMA || ''' = ''L70'' ' ||
        '    AND DAYOFWEEK(CURRENT_DATE()) = 0 ' ||
        '    AND DATEDIFF(DAY, TO_DATE(MAX(HEADER__TIMESTAMP)), CURRENT_DATE()) <= 1 ' ||
        '    THEN ''EXPECTED_GAP - L70 Sunday'' ' ||
 
        -- L70 Monday exception
        '  WHEN ''' || TABLE_SCHEMA || ''' = ''L70'' ' ||
        '    AND DAYOFWEEK(CURRENT_DATE()) = 1 ' ||
        '    AND DATEDIFF(DAY, TO_DATE(MAX(HEADER__TIMESTAMP)), CURRENT_DATE()) <= 2 ' ||
        '    THEN ''EXPECTED_GAP - L70 Monday'' ' ||
 
        -- Standard logic
        '  WHEN TO_DATE(MAX(HEADER__TIMESTAMP)) = CURRENT_DATE() THEN ''LOADED_TODAY - Talend job completed'' ' ||
        '  WHEN DATEDIFF(DAY, TO_DATE(MAX(HEADER__TIMESTAMP)), CURRENT_DATE()) = 1 THEN ''LOADED_YESTERDAY - check TMC if daily table'' ' ||
        '  ELSE ''MISSED - investigate Talend TMC job'' ' ||
 
        'END AS VALIDATION_STATUS ' ||
        'FROM PROD_LANDING.' || TABLE_SCHEMA || '.' || TABLE_NAME || ' ' ||
        'HAVING MAX(HEADER__TIMESTAMP) IS NOT NULL',
        ' UNION ALL '
    )
    INTO :sql_stmt
    FROM PROD_LANDING.INFORMATION_SCHEMA.TABLES
    WHERE TABLE_TYPE = 'BASE TABLE'
      AND TABLE_SCHEMA IN ('L70','DCLM','SAP','DI','LTC','SF','REF') --'SEI'
      AND TABLE_NAME LIKE '%__CT';
 
    sql_stmt := sql_stmt || '
        ORDER BY DAYS_SINCE_LAST DESC NULLS FIRST';
 
    rs := (EXECUTE IMMEDIATE :sql_stmt);
    RETURN TABLE(rs);
 
END;
$$;
Not yet captured:  The top of Step 4.sql (USE ROLE/WAREHOUSE/DATABASE and the full header comment) was cropped out of frame in this screenshot batch — only the section from the header's final lines onward is shown above.
 
A.3 — Step 5.sql: Single-Table Deep-Dive Anomaly Detection
A parameterized, three-method statistical check for one CT table at a time. Shown here as captured, pointed at PROD_LANDING.ING.TZGG5__CT — edit the FROM clause and the two labels in Section 1 (SCHEMA_NM / TABLE_NM) to point at a different table.
Field	Detail
Lookback window	366 days (~1 year) — note: the header comment on daily_ops says "60 days", but the code pulls 366; verify which is intended.
CDC integrity check	Compares UPDATES_TODAY to BEFOREIMAGE_TODAY. Qlik emits a BEFOREIMAGE row for every UPDATE — if they don't match 1:1, that's a CDC integrity problem, not just a volume anomaly.
Method 1 — STDDEV	Flags today's total if it falls outside mean ± 2 standard deviations.
Method 2 — IQR	Flags today's total if it falls outside Q1 − 1.5×IQR / Q3 + 1.5×IQR.
Method 3 — Frequency	Flags if days since last load exceeds the median load gap (DELAYED) or 2× the median gap (MISSED_INTERVAL).
Per-operation flags	Insert, update, and delete counts are each independently checked against their own 2-stddev band.
Overall flag	INVESTIGATE if any of the above trip; REVIEW_NEW_TABLE if fewer than 5 days of history exist; INVESTIGATE_CDC_MISMATCH if the CDC check fails; else OK.

/* ============================================================
   MAIN QUERY
============================================================ */
 
WITH
 
-- ------------------------------------------------------------
-- Step 1: Pull 60 days of daily aggregated operations from the target CT table
--         BEFOREIMAGE rows tracked separately for CDC integrity check only
--         BEFOREIMAGE is NOT counted as a business operation
-- ------------------------------------------------------------
 
daily_ops AS (
    SELECT
        DATE(HEADER__TIMESTAMP)                                              AS LOAD_DT,
        DAYNAME(HEADER__TIMESTAMP)                                           AS DAY_OF_WEEK,
        SUM(CASE WHEN HEADER__OPERATION = 'INSERT'     THEN 1 ELSE 0 END)    AS INS_CNT,
        SUM(CASE WHEN HEADER__OPERATION = 'UPDATE'     THEN 1 ELSE 0 END)    AS UPD_CNT,
        SUM(CASE WHEN HEADER__OPERATION = 'DELETE'     THEN 1 ELSE 0 END)    AS DEL_CNT,
        SUM(CASE WHEN HEADER__OPERATION = 'BEFOREIMAGE' THEN 1 ELSE 0 END)   AS BEFOREIMAGE_CNT,
 
        -- Real business operations only (excludes BEFOREIMAGE)
        SUM(CASE WHEN HEADER__OPERATION IN ('INSERT','UPDATE','DELETE')
                 THEN 1 ELSE 0 END)                                          AS TOT_CNT   -- > CHANGE TARGET TABLE
    FROM PROD_LANDING.ING.TZGG5__CT
    WHERE HEADER__TIMESTAMP >= DATEADD(DAY, -366, CURRENT_DATE())
    GROUP BY DATE(HEADER__TIMESTAMP), DAYNAME(HEADER__TIMESTAMP)
),
 
-- ------------------------------------------------------------
-- Step 2: Gap between consecutive load dates (for cadence detection)
-- ------------------------------------------------------------
 
load_intervals AS (
    SELECT
        LOAD_DT,
        DATEDIFF(DAY, LAG(LOAD_DT) OVER (ORDER BY LOAD_DT), LOAD_DT) AS GAP_DAYS
    FROM daily_ops
    WHERE LOAD_DT < CURRENT_DATE()
),
 
-- ------------------------------------------------------------
-- Step 3: Historical baseline (60 days, excluding today)
-- ------------------------------------------------------------
 
hist_stats AS (
    SELECT
        COUNT(*)                                                     AS DAYS_LOADED_366D,
        AVG(TOT_CNT)                                                 AS HIST_AVG_TOT,
        STDDEV(TOT_CNT)                                              AS HIST_STDDEV_TOT,
        MIN(TOT_CNT)                                                 AS HIST_MIN_TOT,
        MAX(TOT_CNT)                                                 AS HIST_MAX_TOT,
        PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY TOT_CNT)        AS HIST_Q1_TOT,
        PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY TOT_CNT)        AS HIST_MEDIAN_TOT,
        PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY TOT_CNT)        AS HIST_Q3_TOT,
        AVG(INS_CNT)                                                 AS HIST_AVG_INS,
        STDDEV(INS_CNT)                                              AS HIST_STDDEV_INS,
        AVG(UPD_CNT)                                                 AS HIST_AVG_UPD,
        STDDEV(UPD_CNT)                                              AS HIST_STDDEV_UPD,
        AVG(DEL_CNT)                                                 AS HIST_AVG_DEL,
        STDDEV(DEL_CNT)                                              AS HIST_STDDEV_DEL
    FROM daily_ops
    WHERE LOAD_DT < CURRENT_DATE()
),
 
-- ------------------------------------------------------------
-- Step 4: Frequency stats (median is robust against outliers)
-- ------------------------------------------------------------
 
freq_stats AS (
    SELECT
        AVG(GAP_DAYS)                                                AS AVG_GAP_DAYS,
        PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY GAP_DAYS)       AS MEDIAN_GAP_DAYS,
        MAX(GAP_DAYS)                                                AS MAX_GAP_DAYS,
        (SELECT MAX(LOAD_DT) FROM daily_ops)                        AS LAST_LOAD_DT
    FROM load_intervals
    WHERE GAP_DAYS IS NOT NULL
),
 
-- ------------------------------------------------------------
-- Step 5: Today's actual load (includes BEFOREIMAGE for integrity check)
-- ------------------------------------------------------------
 
today_ops AS (
    SELECT
        COALESCE(SUM(INS_CNT), 0)          AS INSERTS_TODAY,
        COALESCE(SUM(UPD_CNT), 0)          AS UPDATES_TODAY,
        COALESCE(SUM(DEL_CNT), 0)          AS DELETES_TODAY,
        COALESCE(SUM(BEFOREIMAGE_CNT), 0)  AS BEFOREIMAGE_TODAY,
        COALESCE(SUM(TOT_CNT), 0)          AS TOTAL_TODAY
    FROM daily_ops
    WHERE LOAD_DT = CURRENT_DATE()
)
 
/* ============================================================
   FINAL OUTPUT
============================================================ */
 
SELECT
    /* SECTION 1: IDENTITY */
    'ING'                                     AS SCHEMA_NM,        -- > CHANGE LABEL
    'TZGG5__CT'                               AS TABLE_NM,         -- > CHANGE LABEL
    CURRENT_DATE()                            AS CHECK_DT,
    DAYNAME(CURRENT_DATE())                   AS CHECK_DAY,
 
    CASE
        WHEN f.MEDIAN_GAP_DAYS IS NULL                     THEN 'UNKNOWN'
        WHEN f.MEDIAN_GAP_DAYS <= 1.5                      THEN 'DAILY'
        WHEN f.MEDIAN_GAP_DAYS BETWEEN 1.5 AND 3           THEN 'BUSINESS_DAILY'
        WHEN f.MEDIAN_GAP_DAYS BETWEEN 3 AND 8             THEN 'WEEKLY'
        WHEN f.MEDIAN_GAP_DAYS BETWEEN 8 AND 35            THEN 'MONTHLY'
        ELSE 'IRREGULAR'
    END                                        AS LOAD_CADENCE,
 
    /* SECTION 2: TODAY'S LOAD (real business operations only) */
    t.INSERTS_TODAY, t.UPDATES_TODAY, t.DELETES_TODAY, t.TOTAL_TODAY,
 
    /* SECTION 3: CDC INTEGRITY CHECK (Qlik BEFOREIMAGE validation) */
    t.BEFOREIMAGE_TODAY,
    CASE
        WHEN t.UPDATES_TODAY = 0 AND t.BEFOREIMAGE_TODAY = 0 THEN 'CDC_OK_NO_UPDATES'
        WHEN t.UPDATES_TODAY = t.BEFOREIMAGE_TODAY           THEN 'CDC_OK'
        ELSE 'CDC_MISMATCH_INVESTIGATE'
    END                                        AS CDC_INTEGRITY_FLAG,
 
    /* SECTION 4: 60-DAY HISTORICAL BASELINE */
    h.DAYS_LOADED_366D,
    ROUND(h.HIST_AVG_TOT, 0) AS HIST_AVG_TOTAL,   ROUND(h.HIST_STDDEV_TOT, 0) AS HIST_STDDEV_TOTAL,
    ROUND(h.HIST_MEDIAN_TOT, 0) AS HIST_MEDIAN_TOTAL,
    h.HIST_MIN_TOT AS HIST_MIN_TOTAL, h.HIST_MAX_TOT AS HIST_MAX_TOTAL,
    ROUND(h.HIST_Q1_TOT, 0) AS HIST_Q1, ROUND(h.HIST_Q3_TOT, 0) AS HIST_Q3,
    ROUND(h.HIST_AVG_INS, 0) AS HIST_AVG_INSERTS, ROUND(h.HIST_AVG_UPD, 0) AS HIST_AVG_UPDATES,
    ROUND(h.HIST_AVG_DEL, 0) AS HIST_AVG_DELETES,
 
    /* SECTION 5: METHOD 1 - STDDEV (mean +/- 2 stddev) */
    GREATEST(ROUND(h.HIST_AVG_TOT - 2 * h.HIST_STDDEV_TOT, 0), 0) AS STDDEV_LOWER_BOUND,
    ROUND(h.HIST_AVG_TOT + 2 * h.HIST_STDDEV_TOT, 0)              AS STDDEV_UPPER_BOUND,
    CASE
        WHEN h.DAYS_LOADED_366D < 5                                             THEN 'INSUFFICIENT_HISTORY'
        WHEN h.HIST_STDDEV_TOT = 0                                              THEN 'STDDEV_NOT_APPLICABLE'
        WHEN t.TOTAL_TODAY < GREATEST(h.HIST_AVG_TOT - 2 * h.HIST_STDDEV_TOT, 0) THEN 'STDDEV_BELOW_NORMAL'
        WHEN t.TOTAL_TODAY > (h.HIST_AVG_TOT + 2 * h.HIST_STDDEV_TOT)           THEN 'STDDEV_ABOVE_NORMAL'
        ELSE 'STDDEV_OK'
    END                                        AS ANOMALY_FLAG_STDDEV,
 
    /* SECTION 6: METHOD 2 - IQR (Q1 - 1.5*IQR, Q3 + 1.5*IQR) */
    GREATEST(ROUND(h.HIST_Q1_TOT - 1.5 * (h.HIST_Q3_TOT - h.HIST_Q1_TOT), 0), 0) AS IQR_LOWER_BOUND,
    ROUND(h.HIST_Q3_TOT + 1.5 * (h.HIST_Q3_TOT - h.HIST_Q1_TOT), 0)              AS IQR_UPPER_BOUND,
    CASE
        WHEN h.DAYS_LOADED_366D < 5                                                          THEN 'INSUFFICIENT_HISTORY'
        WHEN (h.HIST_Q3_TOT - h.HIST_Q1_TOT) = 0                                             THEN 'IQR_NOT_APPLICABLE'
        WHEN t.TOTAL_TODAY < GREATEST(h.HIST_Q1_TOT - 1.5 * (h.HIST_Q3_TOT - h.HIST_Q1_TOT), 0) THEN 'IQR_OUTLIER_LOW'
        WHEN t.TOTAL_TODAY > (h.HIST_Q3_TOT + 1.5 * (h.HIST_Q3_TOT - h.HIST_Q1_TOT))          THEN 'IQR_OUTLIER_HIGH'
        ELSE 'IQR_OK'
    END                                        AS ANOMALY_FLAG_IQR,
 
    /* SECTION 7: METHOD 3 - FREQUENCY */
    ROUND(f.MEDIAN_GAP_DAYS, 1) AS EXPECTED_GAP_DAYS, f.LAST_LOAD_DT,
    DATEDIFF(DAY, f.LAST_LOAD_DT, CURRENT_DATE())                              AS ACTUAL_DAYS_SINCE_LOAD,
    CASE
        WHEN f.MEDIAN_GAP_DAYS IS NULL                                                THEN 'INSUFFICIENT_HISTORY'
        WHEN DATEDIFF(DAY, f.LAST_LOAD_DT, CURRENT_DATE()) > (f.MEDIAN_GAP_DAYS * 2)   THEN 'FREQUENCY_MISSED_INTERVAL'
        WHEN DATEDIFF(DAY, f.LAST_LOAD_DT, CURRENT_DATE()) > f.MEDIAN_GAP_DAYS         THEN 'FREQUENCY_DELAYED'
        ELSE 'FREQUENCY_OK'
    END                                        AS ANOMALY_FLAG_FREQUENCY,
 
    /* SECTION 8: PER-OPERATION FLAGS (same 2-stddev pattern for each) */
    CASE
        WHEN h.DAYS_LOADED_366D < 5 OR h.HIST_STDDEV_INS = 0                        THEN 'INSUFFICIENT_DATA'
        WHEN t.INSERTS_TODAY > (h.HIST_AVG_INS + 2 * h.HIST_STDDEV_INS)             THEN 'INSERTS_SPIKE'
        WHEN t.INSERTS_TODAY < GREATEST(h.HIST_AVG_INS - 2 * h.HIST_STDDEV_INS, 0)  THEN 'INSERTS_BELOW_NORMAL'
        ELSE 'INSERTS_OK'
    END                                        AS FLAG_INSERTS,
    CASE
        WHEN h.DAYS_LOADED_366D < 5 OR h.HIST_STDDEV_UPD = 0                        THEN 'INSUFFICIENT_DATA'
        WHEN t.UPDATES_TODAY > (h.HIST_AVG_UPD + 2 * h.HIST_STDDEV_UPD)             THEN 'UPDATES_SPIKE'
        WHEN t.UPDATES_TODAY < GREATEST(h.HIST_AVG_UPD - 2 * h.HIST_STDDEV_UPD, 0)  THEN 'UPDATES_BELOW_NORMAL'
        ELSE 'UPDATES_OK'
    END                                        AS FLAG_UPDATES,
    CASE
        WHEN h.DAYS_LOADED_366D < 5 OR h.HIST_STDDEV_DEL = 0                        THEN 'INSUFFICIENT_DATA'
        WHEN t.DELETES_TODAY > (h.HIST_AVG_DEL + 2 * h.HIST_STDDEV_DEL)             THEN 'DELETES_SPIKE'
        WHEN t.DELETES_TODAY < GREATEST(h.HIST_AVG_DEL - 2 * h.HIST_STDDEV_DEL, 0)  THEN 'DELETES_BELOW_NORMAL'
        ELSE 'DELETES_OK'
    END                                        AS FLAG_DELETES,
 
    /* SECTION 9: OVERALL FLAG (triggered if ANY check failed) */
    CASE
        WHEN h.DAYS_LOADED_366D < 5                          THEN 'REVIEW_NEW_TABLE'
        WHEN t.UPDATES_TODAY <> t.BEFOREIMAGE_TODAY           THEN 'INVESTIGATE_CDC_MISMATCH'
        WHEN t.TOTAL_TODAY < GREATEST(h.HIST_AVG_TOT - 2 * h.HIST_STDDEV_TOT, 0)
          OR t.TOTAL_TODAY > (h.HIST_AVG_TOT + 2 * h.HIST_STDDEV_TOT)
          OR t.TOTAL_TODAY < GREATEST(h.HIST_Q1_TOT - 1.5 * (h.HIST_Q3_TOT - h.HIST_Q1_TOT), 0)
          OR t.TOTAL_TODAY > (h.HIST_Q3_TOT + 1.5 * (h.HIST_Q3_TOT - h.HIST_Q1_TOT))
          OR DATEDIFF(DAY, f.LAST_LOAD_DT, CURRENT_DATE()) > (f.MEDIAN_GAP_DAYS * 2)
                                                              THEN 'INVESTIGATE'
        ELSE 'OK'
    END                                        AS OVERALL_FLAG
 
FROM hist_stats h
CROSS JOIN today_ops t
CROSS JOIN freq_stats f;
Discrepancy to verify:  The comment above daily_ops says "Pull 60 days", but the WHERE clause and all downstream column names (DAYS_LOADED_366D, etc.) use a 366-day window. Confirm with the author which lookback is intended before relying on the STDDEV/IQR bounds for tables with strong seasonality.
 
A.4 — Step 6.sql: Schema-Wide Anomaly Ranking
The Step 5 logic generalized to every CT table in a schema at once. Runs as an anonymous PL/SQL block: it dynamically builds a UNION ALL across all matching tables, then wraps that in the same distribution-analysis logic, scoped to the schema rather than a single table's own history.
Field	Detail
Parameter	target_schema — set at the top of the block (default 'L70')
Step A	Builds a dynamic UNION ALL across every %__CT base table in the target schema, from PROD_LANDING, over a 366-day window.
Step B	Wraps the union in table_metrics (per-table CDC flag, insert/update/delete %), schema_stats (percentiles, dormant/active table counts), and ranked_tables (activity rank + percentile within schema).
ANOMALY_FLAG_PERCENTILE	PCT_DORMANT / PCT_BOTTOM_10 / PCT_TOP_10 / PCT_NORMAL, relative to the P10/P90 of the whole schema.
ANOMALY_FLAG_STDDEV / _IQR	Same 2-stddev and IQR logic as Step 5, but computed against the schema's distribution rather than one table's own history.
ANOMALY_FLAG_THRESHOLD	Hardcoded absolute floor/ceiling: below 600 records/year = VERY_LOW, above 60,000,000 = VERY_HIGH.
OVERALL_FLAG	INVESTIGATE_DORMANT, INVESTIGATE_CDC, or INVESTIGATE if any percentile/stddev/IQR/threshold check trips; else OK. Results are sorted worst-first.

/* ============================================================
   ANONYMOUS BLOCK - RUNS AS ONE EXECUTION
============================================================ */
 
DECLARE
    target_schema   STRING DEFAULT 'L70';        -- > CHANGE SCHEMA HERE
    union_sql       STRING;
    full_query      STRING;
    res             RESULTSET;
BEGIN
 
    -- ------------------------------------------------------------
    -- Step A: Build dynamic UNION ALL across all CT tables in the schema
    -- ------------------------------------------------------------
 
    SELECT LISTAGG(
        'SELECT ''' || TABLE_SCHEMA || ''' AS SCHEMA_NM, ''' || TABLE_NAME || ''' AS TABLE_NM, ' ||
        'SUM(CASE WHEN HEADER__OPERATION = ''INSERT'' THEN 1 ELSE 0 END) AS INS_366D, ' ||
        'SUM(CASE WHEN HEADER__OPERATION = ''UPDATE'' THEN 1 ELSE 0 END) AS UPD_366D, ' ||
        'SUM(CASE WHEN HEADER__OPERATION = ''DELETE'' THEN 1 ELSE 0 END) AS DEL_366D, ' ||
        'SUM(CASE WHEN HEADER__OPERATION = ''BEFOREIMAGE'' THEN 1 ELSE 0 END) AS BEFOREIMAGE_366D, ' ||
        'SUM(CASE WHEN HEADER__OPERATION IN (''INSERT'',''UPDATE'',''DELETE'') THEN 1 ELSE 0 END) AS TOT_366D, ' ||
        'COUNT(DISTINCT DATE(HEADER__TIMESTAMP)) AS DAYS_LOADED_366D, ' ||
        'MAX(DATE(HEADER__TIMESTAMP)) AS LAST_LOAD_DT ' ||
        'FROM PROD_LANDING.' || TABLE_SCHEMA || '.' || TABLE_NAME || ' ' ||
        'WHERE HEADER__TIMESTAMP >= DATEADD(DAY, -366, CURRENT_DATE())',
        ' UNION ALL '
    ) WITHIN GROUP (ORDER BY TABLE_NAME)
    INTO :union_sql
    FROM PROD_LANDING.INFORMATION_SCHEMA.TABLES
    WHERE TABLE_SCHEMA = UPPER(:target_schema)
      AND TABLE_TYPE = 'BASE TABLE'
      AND TABLE_NAME LIKE '%\_\_CT' ESCAPE '\';
 
    -- ------------------------------------------------------------
    -- Step B: Wrap with the full distribution analysis logic
    -- ------------------------------------------------------------
 
    full_query := '
        WITH table_totals AS (' || :union_sql || '),
 
        table_metrics AS (
            SELECT
                SCHEMA_NM, TABLE_NM,
                INS_366D, UPD_366D, DEL_366D, BEFOREIMAGE_366D, TOT_366D,
                DAYS_LOADED_366D, LAST_LOAD_DT,
                DATEDIFF(DAY, LAST_LOAD_DT, CURRENT_DATE()) AS DAYS_SINCE_LAST_LOAD,
                CASE WHEN DAYS_LOADED_366D > 0 THEN TOT_366D / DAYS_LOADED_366D ELSE 0 END AS AVG_RECS_PER_LOAD_DAY,
                CASE WHEN TOT_366D > 0 THEN ROUND(100.0 * INS_366D / TOT_366D, 1) ELSE 0 END AS INSERT_PCT,
                CASE WHEN TOT_366D > 0 THEN ROUND(100.0 * UPD_366D / TOT_366D, 1) ELSE 0 END AS UPDATE_PCT,
                CASE WHEN TOT_366D > 0 THEN ROUND(100.0 * DEL_366D / TOT_366D, 1) ELSE 0 END AS DELETE_PCT,
                CASE
                    WHEN UPD_366D = 0 AND BEFOREIMAGE_366D = 0 THEN ''CDC_OK_NO_UPDATES''
                    WHEN UPD_366D = BEFOREIMAGE_366D            THEN ''CDC_OK''
                    WHEN BEFOREIMAGE_366D = 0 AND UPD_366D > 0  THEN ''CDC_NO_BEFOREIMAGE''
                    ELSE ''CDC_MISMATCH_INVESTIGATE''
                END AS CDC_INTEGRITY_FLAG
            FROM table_totals
        ),
 
        schema_stats AS (
            SELECT
                AVG(TOT_366D)                                            AS SCHEMA_AVG_TOT,
                STDDEV(TOT_366D)                                         AS SCHEMA_STDDEV_TOT,
                PERCENTILE_CONT(0.10) WITHIN GROUP (ORDER BY TOT_366D)   AS SCHEMA_P10,
                PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY TOT_366D)   AS SCHEMA_Q1,
                PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY TOT_366D)   AS SCHEMA_MEDIAN,
                PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY TOT_366D)   AS SCHEMA_Q3,
                PERCENTILE_CONT(0.90) WITHIN GROUP (ORDER BY TOT_366D)   AS SCHEMA_P90,
                COUNT(*)                                                 AS ACTIVE_TABLES,
                (SELECT COUNT(*) FROM table_metrics)                     AS TOTAL_TABLES_IN_SCHEMA,
                (SELECT COUNT(*) FROM table_metrics WHERE TOT_366D = 0)  AS DORMANT_TABLES
            FROM table_metrics
            WHERE TOT_366D > 0
        ),
 
        ranked_tables AS (
            SELECT
                m.*, s.*,
                RANK() OVER (ORDER BY m.TOT_366D DESC)                                 AS ACTIVITY_RANK,
                ROUND(100.0 * PERCENT_RANK() OVER (ORDER BY m.TOT_366D), 1)            AS PERCENTILE_IN_SCHEMA
            FROM table_metrics m
            CROSS JOIN schema_stats s
        )
 
        SELECT
            SCHEMA_NM, TABLE_NM, CURRENT_DATE() AS CHECK_DT,
            ACTIVITY_RANK, PERCENTILE_IN_SCHEMA, LAST_LOAD_DT, DAYS_SINCE_LAST_LOAD,
 
            TOT_366D AS TOTAL_366D, INS_366D AS INSERTS_366D, UPD_366D AS UPDATES_366D, DEL_366D AS DELETES_366D,
            INSERT_PCT, UPDATE_PCT, DELETE_PCT, DAYS_LOADED_366D,
            ROUND(AVG_RECS_PER_LOAD_DAY, 0) AS AVG_RECS_PER_LOAD_DAY,
 
            BEFOREIMAGE_366D, CDC_INTEGRITY_FLAG,
 
            ROUND(SCHEMA_AVG_TOT, 0) AS SCHEMA_AVG_366D, ROUND(SCHEMA_STDDEV_TOT, 0) AS SCHEMA_STDDEV_366D,
            ROUND(SCHEMA_MEDIAN, 0) AS SCHEMA_MEDIAN_366D, ROUND(SCHEMA_P10, 0) AS SCHEMA_P10, ROUND(SCHEMA_P90, 0) AS SCHEMA_P90,
            ACTIVE_TABLES, DORMANT_TABLES, TOTAL_TABLES_IN_SCHEMA,
 
            CASE
                WHEN TOT_366D = 0           THEN ''PCT_DORMANT''
                WHEN TOT_366D <= SCHEMA_P10 THEN ''PCT_BOTTOM_10''
                WHEN TOT_366D >= SCHEMA_P90 THEN ''PCT_TOP_10''
                ELSE ''PCT_NORMAL''
            END AS ANOMALY_FLAG_PERCENTILE,
 
            GREATEST(ROUND(SCHEMA_AVG_TOT - 2 * SCHEMA_STDDEV_TOT, 0), 0) AS STDDEV_LOWER_BOUND,
            ROUND(SCHEMA_AVG_TOT + 2 * SCHEMA_STDDEV_TOT, 0)             AS STDDEV_UPPER_BOUND,
            CASE
                WHEN TOT_366D = 0                                                       THEN ''STDDEV_DORMANT''
                WHEN SCHEMA_STDDEV_TOT = 0                                               THEN ''STDDEV_NOT_APPLICABLE''
                WHEN TOT_366D < GREATEST(SCHEMA_AVG_TOT - 2 * SCHEMA_STDDEV_TOT, 0)       THEN ''STDDEV_BELOW_SCHEMA_NORM''
                WHEN TOT_366D > (SCHEMA_AVG_TOT + 2 * SCHEMA_STDDEV_TOT)                  THEN ''STDDEV_ABOVE_SCHEMA_NORM''
                ELSE ''STDDEV_OK''
            END AS ANOMALY_FLAG_STDDEV,
 
            GREATEST(ROUND(SCHEMA_Q1 - 1.5 * (SCHEMA_Q3 - SCHEMA_Q1), 0), 0) AS IQR_LOWER_BOUND,
            ROUND(SCHEMA_Q3 + 1.5 * (SCHEMA_Q3 - SCHEMA_Q1), 0)             AS IQR_UPPER_BOUND,
            CASE
                WHEN TOT_366D = 0                                                                THEN ''IQR_DORMANT''
                WHEN (SCHEMA_Q3 - SCHEMA_Q1) = 0                                                 THEN ''IQR_NOT_APPLICABLE''
                WHEN TOT_366D < GREATEST(SCHEMA_Q1 - 1.5 * (SCHEMA_Q3 - SCHEMA_Q1), 0)            THEN ''IQR_OUTLIER_LOW''
                WHEN TOT_366D > (SCHEMA_Q3 + 1.5 * (SCHEMA_Q3 - SCHEMA_Q1))                       THEN ''IQR_OUTLIER_HIGH''
                ELSE ''IQR_OK''
            END AS ANOMALY_FLAG_IQR,
 
            CASE
                WHEN TOT_366D = 0        THEN ''THRESHOLD_DORMANT''
                WHEN TOT_366D < 600      THEN ''THRESHOLD_VERY_LOW''
                WHEN TOT_366D > 60000000 THEN ''THRESHOLD_VERY_HIGH''
                ELSE ''THRESHOLD_NORMAL''
            END AS ANOMALY_FLAG_THRESHOLD,
 
            CASE
                WHEN TOT_366D = 0                                       THEN ''DORMANT''
                WHEN TOT_366D > (SCHEMA_AVG_TOT + 2 * SCHEMA_STDDEV_TOT) THEN ''EXTREME''
                WHEN TOT_366D >= SCHEMA_P90                             THEN ''HIGH''
                WHEN TOT_366D <= SCHEMA_P10                             THEN ''LOW''
                ELSE ''NORMAL''
            END AS ACTIVITY_LEVEL,
 
            CASE
                WHEN TOT_366D = 0
                    THEN ''INVESTIGATE_DORMANT''
                WHEN CDC_INTEGRITY_FLAG = ''CDC_MISMATCH_INVESTIGATE''
                    THEN ''INVESTIGATE_CDC''
                WHEN TOT_366D <= SCHEMA_P10
                  OR TOT_366D >= SCHEMA_P90
                  OR (SCHEMA_STDDEV_TOT > 0 AND TOT_366D < GREATEST(SCHEMA_AVG_TOT - 2 * SCHEMA_STDDEV_TOT, 0))
                  OR (SCHEMA_STDDEV_TOT > 0 AND TOT_366D > (SCHEMA_AVG_TOT + 2 * SCHEMA_STDDEV_TOT))
                  OR ((SCHEMA_Q3 - SCHEMA_Q1) > 0 AND TOT_366D < GREATEST(SCHEMA_Q1 - 1.5 * (SCHEMA_Q3 - SCHEMA_Q1), 0))
                  OR ((SCHEMA_Q3 - SCHEMA_Q1) > 0 AND TOT_366D > (SCHEMA_Q3 + 1.5 * (SCHEMA_Q3 - SCHEMA_Q1)))
                  OR TOT_366D < 600
                  OR TOT_366D > 60000000
                    THEN ''INVESTIGATE''
                ELSE ''OK''
            END AS OVERALL_FLAG
 
        FROM ranked_tables
        ORDER BY
            CASE
                WHEN TOT_366D = 0 THEN 1
                WHEN CDC_INTEGRITY_FLAG = ''CDC_MISMATCH_INVESTIGATE'' THEN 2
                WHEN TOT_366D <= SCHEMA_P10 THEN 3
                WHEN TOT_366D >= SCHEMA_P90 THEN 4
                ELSE 5
            END,
            TOT_366D DESC';
 
    -- ------------------------------------------------------------
    -- Step C: Execute the full query and return results
    -- ------------------------------------------------------------
 
    res := (EXECUTE IMMEDIATE :full_query);
    RETURN TABLE(res);
 
END;
 
A.5 — Step 8.sql: Missing Table Load Detection & Freshness
Checks a single CT table's load cadence directly — no historical baseline table needed, it derives the expected gap from the table's own load history. Shown here pointed at PROD_LANDING.ING.TZRAE__CT.
Field	Detail
Lookback window	365 days
EXPECTED_WINDOW_DAYS	1.5 × the largest historical gap ever seen between two loads for this table
LOADED_TODAY	Last load date is today
EXPECTED_GAP	Days since last load is within the expected window
UNEXPECTED_MISS	Days since last load is within 3× the expected window, but past it
STOPPED_LOADING	Days since last load exceeds 3× the expected window
MISSED_EXPECTED_LOAD_FLAG	Y once past the expected window (early warning)
STOPPED_LOADING_FLAG	Y once past 3× the expected window (escalation trigger)

/* ============================================================
   Step 8 - Missing Table Load Detection & Data Freshness Monitoring
============================================================ */
 
WITH LOAD_DATES AS
(
    SELECT DISTINCT
        TO_DATE(HEADER__TIMESTAMP) AS LOAD_DATE
    FROM PROD_LANDING.ING.TZRAE__CT
    WHERE HEADER__TIMESTAMP IS NOT NULL
      AND TO_DATE(HEADER__TIMESTAMP) >= DATEADD(DAY, -365, CURRENT_DATE())
),
 
GAPS AS
(
    SELECT
        LOAD_DATE,
        DATEDIFF(
            DAY,
            LAG(LOAD_DATE) OVER (ORDER BY LOAD_DATE),
            LOAD_DATE
        ) AS GAP_DAYS
    FROM LOAD_DATES
),
 
TABLE_STATS AS
(
    SELECT
        COUNT(*)       AS ACTIVE_LOAD_DAYS,
        MAX(LOAD_DATE) AS LAST_LOAD_DATE,
        MAX(GAP_DAYS)  AS MAX_GAP_DAYS
    FROM GAPS
),
 
ANALYSIS AS
(
    SELECT
        ACTIVE_LOAD_DAYS,
        LAST_LOAD_DATE,
        MAX_GAP_DAYS,
        ROUND(MAX_GAP_DAYS * 1.5, 2) AS EXPECTED_WINDOW_DAYS,
        DATEDIFF(
            DAY,
            LAST_LOAD_DATE,
            CURRENT_DATE()
        ) AS DAYS_SINCE_LAST_LOAD
    FROM TABLE_STATS
)
 
SELECT
    ACTIVE_LOAD_DAYS,
    LAST_LOAD_DATE,
    MAX_GAP_DAYS,
    EXPECTED_WINDOW_DAYS,
    DAYS_SINCE_LAST_LOAD,
 
    CASE
        WHEN LAST_LOAD_DATE = CURRENT_DATE()
            THEN 'LOADED_TODAY'
        WHEN DAYS_SINCE_LAST_LOAD <= EXPECTED_WINDOW_DAYS
            THEN 'EXPECTED_GAP'
        WHEN DAYS_SINCE_LAST_LOAD <= EXPECTED_WINDOW_DAYS * 3
            THEN 'UNEXPECTED_MISS'
        ELSE 'STOPPED_LOADING'
    END AS LOAD_STATUS,
 
    CASE
        WHEN DAYS_SINCE_LAST_LOAD > EXPECTED_WINDOW_DAYS
            THEN 'Y'
        ELSE 'N'
    END AS MISSED_EXPECTED_LOAD_FLAG,
 
    CASE
        WHEN DAYS_SINCE_LAST_LOAD > EXPECTED_WINDOW_DAYS * 3
            THEN 'Y'
        ELSE 'N'
    END AS STOPPED_LOADING_FLAG
 
FROM ANALYSIS;
 
Appendix B — Escalation Quick Reference
Trigger	Action
Anomaly found in Steps 6–9	Document in "Observations/Anomalies" OR escalate to Siddesh immediately to confirm if an incident is needed.
Behavior needs a second opinion	Contact Joshua Burns, Maribeth Daus, John Steinbeck, or Devin Jones.
Confirmed expected behavior	Log in the "Observations/Anomalies" sheet — no incident.
Confirmed real issue	Create an incident per KofC – Incident Management Process v5.docx; log in "Snowflake Prod Incidents" sheet.
Fix ready for Test	Contact Peter (Peter Pirro) to open an ADO ticket; tag him in Daily Modernization – Daily Build and Bug Scrum chat.
Test → UAT → UAT2 → Prod	Raise an RFC (Change Request) at each promotion step; Peter assigns Lionel or Sam to test.
 
Appendix C — Open Items for the Next Update
The following were referenced during this update but their SQL bodies were not captured and are not documented above. Do not assume their logic mirrors Steps 2–8 above — capture and add them before relying on this document for those checks.
•	Step 9.sql — FMC schema / Airflow DAG monitoring in the Raw Vault (Guide Step 9).
•	FMC_Airflow_Dag.sql — in the Monitoring queries folder; likely related to or supporting Step 9.
•	Monitoring query.sql — per-table record count query used during daily monitoring.
•	Validation query.sql — general validation query referenced alongside Monitoring query.sql.
Also verify:  The 60-day vs. 366-day discrepancy noted in Appendix A.3, and whether the top of Step 4.sql (role/warehouse/database, full header) matches Step 2.sql's pattern exactly.
