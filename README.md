Hi team,
 
We've traced a duplicate-record issue affecting this quarter's IFRS17 actuarial
reporting back to KCDWHPRC.T8LPUA. 19 business keys currently have more than
one row simultaneously flagged as the "current" version (CRNT_REC_IND = 'Y'),
which should never happen in a Type-2 history table.
 
We've ruled out a design defect in the ETL job (T8LPUA_CDC_LOAD) itself —
the comparison key matches the table's own unique index, and the mechanism
correctly expires prior versions across years of repeated reprocessing for
the same policies. In fact, for one example key we traced its FULL history
back to 2008 (41 load events) and found only ONE failed transition in the
entire history:
 
  - RPT_PRCES_STATS_ID 4291273, ran 2026-02-14: correctly marked current
  - RPT_PRCES_STATS_ID 4357177, ran 2026-05-09: FAILED to expire the
    prior row — both rows are now flagged Y
 
This points to something specific that happened DURING that one execution
(4357177, ~2026-05-09 10:02 AM) rather than a code-level defect — for
example a failed run that was resubmitted without fully re-running its
expire step, or a partial commit where the expire UPDATE didn't commit
before a later INSERT did.
 
We don't have access to pull job execution history that far back (our local
Designer Monitor tab shows nothing for that date, likely outside the
retention window). Could someone with Job Server / Management Console
access please check:
 
  1. The execution detail/trace log for T8LPUA_CDC_LOAD on 2026-05-09
     around 10:00-10:05 AM (RPT_PRCES_STATS_ID 4357177)
  2. Whether that run shows as a first attempt or a resubmit/recovery of
     an earlier failed execution
  3. Whether "Recover from last failed execution" was enabled for that run
  4. Any warnings/errors at the Table_Comparison or History_Preserving
     transform steps specifically
 
Happy to hop on a call to walk through the full investigation — we have
the complete evidence chain documented, including the specific queries
and results that ruled out the other explanations we tested first.
 
Thanks,
Lokesh
