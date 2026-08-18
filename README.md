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

Using HEADER__TIMESTAMP, identify all CT tables loaded today from both Qlik and Talend. 

For each identified table, validate that records were loaded for the current day. 

Verify today's load volume aligns with historical trends by comparing to previous runs. 

Assess whether today's load volume falls within the expected range using statistical measures (e.g. standard deviation). 

Flag the load if it deviates by roughly 2–3 standard deviations from historical norms for further investigation. 

Essentially, confirm there is no unusual spike or drop in the data load, and that load volume and frequency remain consistent with typical behavior. 

Run:  Step 5.sql (single-table deep dive — edit the target table before running) — see Appendix A.3. 

Step 6 — Identify Outlier Tables by Insert/Update/Delete Counts 

Identify CT tables loaded today by Qlik or Talend that have the highest and lowest counts of inserted, updated, and deleted records, and compare them against historical trends to flag potential outliers. 

Sort and Shortlist Tables 

Sort CT tables loaded by Qlik or Talend by the number of records inserted, updated, or deleted today (ascending order). 

Focus on tables with the lowest and highest number of records inserted/updated/deleted today, as both can indicate potential issues. Investigate these further. 

Compare Against Historical Trends 

Analyze historical volumes of records inserted, updated, or deleted per day for each table to understand typical behavior. 

Establish expected ranges or patterns based on past load trends. 

Compare today's insert/update/delete counts against these expected ranges to identify deviations. 

Use statistical measures such as standard deviation to determine whether today's load deviates significantly (e.g. by 2–3 standard deviations) from historical norms; if so, investigate the underlying cause. 

Flag Potential Outliers 

Highlight tables where today's insert/update/delete counts significantly deviate from historical norms. 

Example: If a table typically has 200–300 records inserted/updated/deleted daily but only shows ~100 today, flag it for review. 

Use statistical measures such as interquartile range (IQR) to define upper and lower bounds for outlier detection, and standard deviation to assess whether today's load deviates significantly (e.g. 2–3 standard deviations) from historical norms; if it does, investigate the underlying cause. 

Make a Note of Your Findings 

Note these for follow-up, with the intent to either document them in the "Observations/Anomalies" section of the Snowflake Issues Tracker or raise them as an incident, as appropriate. Escalate your findings to Siddesh immediately to confirm if an Incident needs to be created. 

Run:  Step 6.sql (schema-wide — set target_schema before running) — see Appendix A.4. 

Step 7 — Reconcile Discrepancies Against Source (AQT) 

If any discrepancies are identified in Step 6, validate the data load by querying the source system using AQT and comparing record counts with the corresponding target tables to ensure data consistency. 

Confirm with Source System 

Use AQT (Advanced Query Tool) to directly query the source system tables. 

Validate that the data present in the source aligns with what is expected for the load to the target. 

Compare Record Counts 

Run record count queries on both source tables (via AQT) and target/loaded tables (snapshot tables). 

Ensure counts match or fall within acceptable thresholds. 

Perform Detailed Validation (if counts don't match) 

Identify missing, duplicate, or mismatched records. 

Make a Note of Your Findings 

Capture any differences, including record count variances and data inconsistencies. 

Note whether the issue originates from the source, transformation logic, or load process. 

Escalate your findings to Siddesh immediately to confirm if an Incident needs to be created. 

Step 8 — Review CT Tables Not Loaded Today 

Review all CT tables that were not loaded today to determine whether this is expected behavior; for any unexpected gaps, investigate the root cause to understand why those tables were not loaded. 

Identify Missing Loads 

Generate a list of CT tables with no records inserted, updated, or deleted today. 

Validate Expected Behavior 

Confirm whether these tables are not expected to load daily (e.g. based on schedule, load frequency, dependencies, or business logic). Use the validation query to check load frequency and schedule. 

Investigate Unexpected Gaps 

For tables that should have been loaded, investigate the root cause, including potential upstream data issues, Qlik replication or Talend job failures, as well as pipeline dependencies or configuration problems. 

Make a Note of Your Findings 

Note these for follow-up; document in "Observations/Anomalies" or raise as an incident. Escalate to Siddesh immediately. 

Run:  Step 8.sql (single table — edit the target table before running) — see Appendix A.5. 

Step 9 — Review FMC Schema / Airflow DAG Loads (Raw Vault) 

Review the FMC schema in the Raw Vault to monitor Airflow DAG loads from Prod Landing, focusing on ING, FRAT_ING, and FRATDB DAGs, to ensure successful runs and that record counts align with historical trends. 

Validate DAG Execution Status 

Focus on Airflow DAGs for ING, FRAT_ING, and FRATDB (typically prefixed with "FMC_3D"). 

Confirm that all scheduled DAGs have completed successfully and that data has been loaded as expected. 

Identify any failed or long-running DAGs with the help of the FMC schema. 

Validate Data Load and Performance 

Check the number of records processed by each DAG and verify it matches the volumes normally observed for that load. 

Confirm data is flowing correctly from Prod Landing to the Raw Vault. 

Identify long-running DAGs, particularly those exceeding typical runtimes (e.g. ~2–3 standard deviations above normal). 

Compare Against Historical Trends 

Analyze historical load volumes for each DAG. 

Compare today's load volumes against expected volumes derived from historical trends. 

Compare the current runtime to historical runtimes to ensure it falls within the normal expected range for the DAG. 

Flag and Investigate Anomalies 

Identify potential anomalies in records loaded, DAG runtimes, and DAG execution status (e.g. failures or missed runs) using the FMC schema in the Raw Vault. 

Investigate potential causes such as ingestion delays, pipeline failures, upstream data issues, or unexpected changes in source data. 

Make a Note of Your Findings 

Make note of any anomalies or irregularities for later review to determine whether they should be logged as observations/anomalies or escalated as an incident. Escalate your findings to Siddesh immediately. 

Run:  Step 9.sql — not yet captured in this document. See Appendix C. 

Steps 10–12 — Escalate & Confirm 

Escalate any identified or unusual issues immediately and confirm with members of the team whether the behavior is expected. Update the Snowflake Issues Tracker and raise an incident accordingly. 

Procedure 

Consolidate the findings you made a note of in Steps 6, 7, 8, and 9. 

If any anomalies, inconsistencies, or potential issues are identified, escalate the findings to your manager (Siddesh) with all relevant details. 

Reach out to one of the following team members to validate whether the observed behavior is expected: 

Joshua Burns 

Maribeth Daus 

John Steinbeck 

Devin Jones 

Based on the feedback received: 

If the behavior is confirmed as expected — document the observations in the "Observations/Anomalies" sheet. 

If the behavior is confirmed as an issue — create an incident following the standard incident management process (KofC – Incident Management Process v5.docx) and record the details in the "Snowflake Prod Incidents" sheet. 

Step 13 — Develop the Fix (Dev Environment) 

Develop and implement the fix in the Dev environment. Thoroughly test it to confirm it is functioning as expected. 

Steps 14–19 — Promote the Fix to Production 

Once the fix is thoroughly tested and confirmed to be working as expected, follow the steps below. Keep Siddesh informed throughout the entire process. 

1. Create a Service Request for deployment to Test 

Contact Peter to create an ADO ticket for testers. 

Peter will assign the ticket to Lionel or Sam for testing in the Test and UAT environments. 

Send the ticket number to the Daily Modernization – Daily Build and Bug Scrum group chat and tag Peter Pirro. Include a brief one-line description of the task and what it resolves. 

Create the Service Request for the promote to Test first. Once the fix is deployed and tested, raise the Service Request to promote it from Test → UAT. Repeat the tag/description step above. Once the fix is promoted to UAT, coordinate with the tester to test the fix. 

2. Create an RFC for UAT → UAT2 Deployment 

After successful completion of the previous step, create an RFC (Change Request) for deployment from UAT to UAT2, following the same tag/description process as above. 

3. UAT2 Testing 

Peter will assign Sam or Lionel to perform testing in the UAT2 environment. After the promote to UAT2, coordinate with the tester to test the fix. 

4. Promote to Prod 

If testing in UAT2 is successful, create an RFC (Change Request) to promote the changes from UAT2 to Prod, following the same process used for the UAT → UAT2 promotion — the only difference is that testing will be performed in Prod. 

 

Appendix A — SQL Script Reference 

Full scripts, transcribed as-built from the workspace. Each script is a stored, reusable query in the Drill down queries folder — several are parameterized and must be edited (schema, table name) before running against a new target. 

A.1 — Step 2.sql: Qlik CT Table Replication Validation (Q-QLIK) 

Field 

Detail 

Purpose 

Confirm Qlik-loaded CT tables show a recent HEADER__TIMESTAMP, indicating CDC replication is active. 

Scanned schemas 

ING, FRATDB, RPLUS, FRAT_ING (only FRATDB is currently un-commented / active) 

Source database 

DEV_LANDING (per USE DATABASE at top of script) 

FRESH 

Last HEADER__TIMESTAMP within 4 hours 

STALE_TODAY 

Updated today, but 4–24 hours old 

MISSED_TODAY 

No updates today 


