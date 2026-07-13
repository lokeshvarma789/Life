Subject: IFRS17 Q2 Close — Data Validation Update: Multi-Record Policies Confirmed as Expected

Hi team,

Quick update on the data validation review currently underway as part of the Q2 2026 IFRS17 close.

During validation, we identified cases where individual policies appeared to have more than one record in the policy valuation output. We traced each of these back to source in detail, and can confirm these are expected, not data quality issues:

- Some policies are serviced through two different source systems (Ingenium and Life70). Each source correctly contributes its own record for the same policy/company code — this shows up as "multiple rows" but reflects two genuinely different source records, not a duplicate.
- Separately, some policies show both an active (in-force) record and a distinct termination record. A policy's active cash value and its termination cash value are tracked as separate records by design, and termination records correctly show no cash value amount.

Both patterns are valid, expected business scenarios. They do not indicate any data quality concern with the underlying process.

Based on this, we're clearing this item and confirming the DBA team can continue with the scheduled quarterly script execution. April's data has completed successfully, and the May script is currently running; June will follow once May completes.

Separately — and unrelated to the above — we identified one confirmed data issue in a different underlying table during this same review. That item remains open and is being tracked separately with the DBA team; it does not affect the timeline above. We'll follow up once we have more to share there.

Happy to answer any questions in the meantime.

Thanks,
Lokesh

Subject: IFRS17 Q2 Close — Data Check Update
Hi team, quick update — we looked into some policies that appeared to have more than one record, and confirmed it's expected, not an error. It happens when a policy has data from two different source systems, or has a separate termination record alongside its active one. Both are normal and not a data issue. Based on this, the DBA team can go ahead with the May and June script runs — April is already done. Separately, we found one unrelated data issue in another table that we're still looking into with DBA; that one doesn't affect this timeline. Happy to answer any questions.
Thanks, Lokesh
