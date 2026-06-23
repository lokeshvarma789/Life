Hi Mary Beth and Josh,

We've been investigating INC0054877 / BUG 44624 (PROD_DV.INFO_MART.D_MEMBER includes deleted members and has duplicates). The duplicate piece was addressed in the previous change, but the deleted-members portion of the ticket is still open and we'd like your input before we proceed.

What we found:

- D_MEMBER currently sources directly from BUSINESS_VAULT.S_STD_MEMBER and does not apply any filter on MEMBER_DELETE_FLAG. The flag is set correctly upstream (3,366 members flagged as deleted in PROD), but nothing in the view excludes them.
- The previous dedup fix was applied at the BR_STD_MEMBER layer, which D_MEMBER does not pull from, so the delete flag issue was not addressed.
- D_CLIENT has the same structural gap (no filter on CLIENT_DELETE_FLAG), though no client records are currently flagged, so it isn't leaking today.
- UAT2 has the same DDL as PROD, so this is a missing-logic issue rather than a deployment gap.

Questions for you:

1. Should deleted members be excluded entirely from D_MEMBER, or should they remain in the dimension with the delete flag exposed so consumers can filter on their side?
2. If excluded, is the correct layer to apply the filter at the INFO_MART view (D_MEMBER), or would you prefer it pushed upstream into the business rules layer for consistency across all downstream consumers?
3. Should we apply the same approach preventively to D_CLIENT, given the identical structural gap?

For context, excluding deleted members would remove approximately 3,366 records from D_MEMBER, which may affect downstream reports joining to this dimension.

Our recommendation is to add the filter directly at the D_MEMBER view level for a minimal, contained change, and apply the same pattern to D_CLIENT. Happy to walk through the investigation findings on a quick call if that's easier.

Please let us know your preference so we can proceed with the change request.

Thanks,
[Your name]
