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



A.2 — Step 4.sql: Talend CT Table Load Validation 

Field 

Detail 

Purpose 

Confirm Talend-loaded CT tables show a recent HEADER__TIMESTAMP, indicating the scheduled load completed. 

Scanned schemas 

L70, DCLM, SAP, DI, LTC, SF, REF (SEI currently commented out) 

Source database 

PROD_LANDING 

L70 Sunday exception 

If today is Sunday and L70's last load is ≤1 day old, flagged EXPECTED_GAP instead of MISSED 

L70 Monday exception 

If today is Monday and L70's last load is ≤2 days old, flagged EXPECTED_GAP instead of MISSED 

Standard logic 

LOADED_TODAY / LOADED_YESTERDAY (check TMC if daily table) / MISSED — investigate Talend TMC job 

A.3 — Step 5.sql: Single-Table Deep-Dive Anomaly Detection 

A parameterized, three-method statistical check for one CT table at a time. Shown here as captured, pointed at PROD_LANDING.ING.TZGG5__CT — edit the FROM clause and the two labels in Section 1 (SCHEMA_NM / TABLE_NM) to point at a different table. 

Field 

Detail 

Lookback window 

366 days (~1 year) — note: the header comment on daily_ops says "60 days", but the code pulls 366; verify which is intended. 

CDC integrity check 

Compares UPDATES_TODAY to BEFOREIMAGE_TODAY. Qlik emits a BEFOREIMAGE row for every UPDATE — if they don't match 1:1, that's a CDC integrity problem, not just a volume anomaly. 

Method 1 — STDDEV 

Flags today's total if it falls outside mean ± 2 standard deviations. 

Method 2 — IQR 

Flags today's total if it falls outside Q1 − 1.5×IQR / Q3 + 1.5×IQR. 

Method 3 — Frequency 

Flags if days since last load exceeds the median load gap (DELAYED) or 2× the median gap (MISSED_INTERVAL). 

Per-operation flags 

Insert, update, and delete counts are each independently checked against their own 2-stddev band. 

Overall flag 

INVESTIGATE if any of the above trip; REVIEW_NEW_TABLE if fewer than 5 days of history exist; INVESTIGATE_CDC_MISMATCH if the CDC check fails; else OK. 

A.4 — Step 6.sql: Schema-Wide Anomaly Ranking 

The Step 5 logic generalized to every CT table in a schema at once. Runs as an anonymous PL/SQL block: it dynamically builds a UNION ALL across all matching tables, then wraps that in the same distribution-analysis logic, scoped to the schema rather than a single table's own history. 
Appendix B — Escalation Quick Reference 

Trigger 

Action 

Anomaly found in Steps 6–9 

Document in "Observations/Anomalies" OR escalate to Siddesh immediately to confirm if an incident is needed. 

Behavior needs a second opinion 

Contact Joshua Burns, Maribeth Daus, John Steinbeck, or Devin Jones. 

Confirmed expected behavior 

Log in the "Observations/Anomalies" sheet — no incident. 

Confirmed real issue 

Create an incident per KofC – Incident Management Process v5.docx; log in "Snowflake Prod Incidents" sheet. 

Fix ready for Test 

Contact Peter (Peter Pirro) to open an ADO ticket; tag him in Daily Modernization – Daily Build and Bug Scrum chat. 

Test → UAT → UAT2 → Prod 

Raise an RFC (Change Request) at each promotion step; Peter assigns Lionel or Sam to test. 

