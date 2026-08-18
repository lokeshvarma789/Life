A.4 — Step 6.sql: Schema-Wide Anomaly Ranking 

The Step 5 logic generalized to every CT table in a schema at once. Runs as an anonymous PL/SQL block: it dynamically builds a UNION ALL across all matching tables, then wraps that in the same distribution-analysis logic, scoped to the schema rather than a single table's own history. 

Field 

Detail 

Parameter 

target_schema — set at the top of the block (default 'L70') 

Step A 

Builds a dynamic UNION ALL across every %__CT base table in the target schema, from PROD_LANDING, over a 366-day window. 

Step B 

Wraps the union in table_metrics (per-table CDC flag, insert/update/delete %), schema_stats (percentiles, dormant/active table counts), and ranked_tables (activity rank + percentile within schema). 

ANOMALY_FLAG_PERCENTILE 

PCT_DORMANT / PCT_BOTTOM_10 / PCT_TOP_10 / PCT_NORMAL, relative to the P10/P90 of the whole schema. 

ANOMALY_FLAG_STDDEV / _IQR 

Same 2-stddev and IQR logic as Step 5, but computed against the schema's distribution rather than one table's own history. 

ANOMALY_FLAG_THRESHOLD 

Hardcoded absolute floor/ceiling: below 600 records/year = VERY_LOW, above 60,000,000 = VERY_HIGH. 

OVERALL_FLAG 

INVESTIGATE_DORMANT, INVESTIGATE_CDC, or INVESTIGATE if any percentile/stddev/IQR/threshold check trips; else OK. Results are sorted worst-first. 

A.5 — Step 8.sql: Missing Table Load Detection & Freshness 

Checks a single CT table's load cadence directly — no historical baseline table needed, it derives the expected gap from the table's own load history. Shown here pointed at PROD_LANDING.ING.TZRAE__CT. 

Field 

Detail 

Lookback window 

365 days 

EXPECTED_WINDOW_DAYS 

1.5 × the largest historical gap ever seen between two loads for this table 

LOADED_TODAY 

Last load date is today 

EXPECTED_GAP 

Days since last load is within the expected window 

UNEXPECTED_MISS 

Days since last load is within 3× the expected window, but past it 

STOPPED_LOADING 

Days since last load exceeds 3× the expected window 

MISSED_EXPECTED_LOAD_FLAG 

Y once past the expected window (early warning) 

STOPPED_LOADING_FLAG 

Y once past 3× the expected window (escalation trigger) 

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

