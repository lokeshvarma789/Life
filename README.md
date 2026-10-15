You are a senior data engineer with deep experience in WebFOCUS (.fex procedures),
IBM DB2 (LUW/Linux), SAP BODS CDC loads, and type-2 history tables
(CURR_IND / effective-dated rows). Help me find the root cause of a reporting
discrepancy. Be precise. Do not guess: if something cannot be determined from
what I give you, say so and tell me what query or file you need.

=====================================================================
BACKGROUND
=====================================================================
- Report: AV111 (WebFOCUS), runs at month end. The Sept 30, 2026 run returned
  Canada data but no USA data, because of a bad vendor input file.
- The vendor sent a corrected file. The business asked us to rerun the report
  for the Sept 30 period only.
- USA data comes from one table loaded by a BODS CDC job (inserts new records
  only, so a rerun should not create duplicates).
- Two other tables feed the report: <<D8LTL>> and <<D8GPO>>. They are weekly /
  bi-weekly tables, so they now contain October data that must NOT appear in a
  Sept 30 report.
- My fix:
    1. Created snapshot tables in ProdDM: <<SNAP_D8LTL>> and <<SNAP_D8GPO>>,
       holding only the Sept period, using CURR_IND = 'Y' plus a date filter.
    2. Copied the production .fex and used find/replace to point the original
       table names at the snapshot tables. My intent was to change ONLY table
       names. Nothing else.
    3. Ran the modified .fex.
- Result: USA data now appears, but there are FEWER eligibility-related
  records than expected.
- What I already checked: base table logic and snapshot logic return matching
  results. The new data has ELIG_CD = '8', and the eligibility mapping table
  (sourced from a data mart) has no row for code 8.

=====================================================================
INPUTS
=====================================================================
[A] ORIGINAL production .fex (unchanged):
<<paste full original .fex>>

[B] MODIFIED .fex (the one I ran):
<<paste full modified .fex>>

[C] SQL I used to create the snapshot tables:
<<paste CREATE TABLE / INSERT ... SELECT statements>>

[D] Base table DDL or column list for D8LTL / D8GPO (especially CURR_IND,
    effective start/end dates, update timestamps, key columns):
<<paste DDL or DESCRIBE output>>

[E] Eligibility mapping table name and columns:
<<table name + columns>>

[F] Counts I observed (optional):
<<expected vs actual eligibility counts, ELIG_CD distribution, etc.>>

=====================================================================
TASK 1: Verify the .fex change was table-name-only
=====================================================================
Compare [A] and [B] line by line and report EVERY difference, grouped as:
  a) Expected: table/segment/master-file name substitutions only.
  b) Unexpected: anything else, including but not limited to:
     - WHERE / IF / WHERE TOTAL filter changes
     - Find/replace side effects: a table name that was also a substring of a
       field name, alias, DEFINE/COMPUTE name, or another table name
     - JOIN changes: JOIN type, keys, AS names, or JOIN CLEAR
     - DEFINE / COMPUTE logic
     - Amper variables (&RPTDT, &YYMM, -SET, -DEFAULT, -IF) and date
       parameters
     - SQL passthru blocks (SQL DB2 ... END)
     - Master file / access file references, or schema/owner prefix changes
     - Hold files, TABLE FILE targets, ON TABLE options
  For each unexpected difference: give line numbers, old vs new text, and
  whether it could reduce eligibility record counts.

Also confirm:
  - Every original table reference was replaced: list any that were missed.
  - No occurrence was replaced that should not have been: list each one.
  - Whether the snapshot tables need their own WebFOCUS master/access files,
    or whether the .fex hits DB2 directly through SQL passthru. If master
    files are needed and the snapshot uses a different column structure,
    flag it.

=====================================================================
TASK 2: Check whether CURR_IND = 'Y' is the wrong snapshot logic
=====================================================================
Analyze [C] against [D] and answer:
  1. Does CURR_IND = 'Y' return the state of each record AS OF Sept 30, or
     the state AS OF TODAY? Explain with a concrete example: a policy updated
     on Oct 5 and what happens to its Sept-valid row.
  2. Combined with my date filter, which records would be wrongly EXCLUDED
     (valid on 9/30 but superseded later), and which would be wrongly
     INCLUDED (post-9/30 state leaking in)?
  3. Could ELIG_CD = '8' in the snapshot be post-9/30 data that leaked in
     because of CURR_IND = 'Y'?
  4. Write the corrected point-in-time snapshot SQL for DB2 LUW, two
     versions:
       a) If the table has effective start/end dates:
          EFF_STRT_DT <= '2026-09-30'
          AND (EFF_END_DT > '2026-09-30' OR EFF_END_DT IS NULL)
       b) If history is tracked only by an update timestamp:
          latest row per business key where UPDT_TS <= '2026-09-30-23.59.59'
          (use ROW_NUMBER() in a subquery, because DB2 does not support
          QUALIFY)
     Use the real column names from [D].

=====================================================================
TASK 3: Write validation queries (DB2 LUW syntax)
=====================================================================
Give me ready-to-run queries for:
  Q1. ELIG_CD distribution: base table vs. my snapshot vs. the corrected
      point-in-time logic, side by side.
  Q2. When ELIG_CD = '8' first appeared: MIN/MAX of effective date and update
      timestamp in the base table.
  Q3. Keys valid on 9/30 that are MISSING from my snapshot, and keys in my
      snapshot that were NOT valid on 9/30.
  Q4. Snapshot rows with no match in the eligibility mapping table, by
      ELIG_CD. Join with TRIM() on both sides, and also show LENGTH() and
      HEX() of the codes so CHAR padding or '8' vs '08' format mismatches
      are visible.
  Q5. Whether the mapping table has its own CURR_IND or effective dating
      that could hide code 8.
  Q6. Reconciliation: does (expected count - actual count) equal the number
      of rows dropped by the mapping join? If not, quantify the remaining
      gap.

=====================================================================
TASK 4: Conclusion and recommendation
=====================================================================
Based on everything above, tell me:
  - The most likely root cause, ranked if there is more than one:
    (a) CURR_IND snapshot logic, (b) unintended .fex change,
    (c) ELIG_CD 8 missing from the mapping, (d) data type/format mismatch,
    (e) other.
  - Whether ELIG_CD 8 is a genuine new business code (needs a data-mart
    change request; we must not add the mapping ourselves) or a symptom of
    the snapshot pulling post-period data.
  - Exact next steps, in order.
  - A 3-4 sentence update I can paste into the incident ticket, written in
    plain language for a non-technical business reader.

=====================================================================
CONSTRAINTS
=====================================================================
- Do NOT suggest modifying BODS jobs, the shared TMAS control table, or
  production base tables.
- Do NOT suggest changing the JOIN to an outer join just to make counts
  match.
- Do NOT suggest adding a mapping row for ELIG_CD 8 as a fix. Flag it as a
  business/data-mart decision.
- Any fix must be limited to the snapshot tables and the copied .fex, and be
  fully reversible.
- Prefer DB2 LUW syntax. Point out any syntax that differs on DB2 z/OS.
