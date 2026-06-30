
Short Description:
UMOD: Add MEMBER_DELETE_FLAG column to PROD_DV.INFO_MART.D_MEMBER view

---

Description:

Type of Change: Application/Software
Sub Type: Snowflake
Related Incident: INC0054877
Related Bug: BUG 44624

Background:
As part of the investigation for INC0054877, it was identified that the MEMBER_DELETE_FLAG column is currently commented out in the D_MEMBER view definition (PROD_DV.INFO_MART.D_MEMBER). Following a review call on 06/29/2026 with Josh Burns, John Steinbeck, and Liam Foley, the team has agreed to uncomment and expose this column so that business consumers can apply their own filter (WHERE MEMBER_DELETE_FLAG <> 'Y') as needed in their queries.

Change Required:
- Uncomment MEMBER_DELETE_FLAG column in the D_MEMBER view definition
- Add any additional columns confirmed by Liam Foley (pending EOD 06/29/2026)
- No data is being deleted or modified — this is a view-only change

Promotion Path:
DEV >> TEST >> UAT2 >> PROD

Impact:
- Low risk — view-only change, no upstream logic modified
- Consumers will now see MEMBER_DELETE_FLAG in D_MEMBER
- No existing columns are being removed or renamed
- Downstream reports will not break — only a new column is being added

Rollback Plan:
- Re-comment the column in the view definition if any issues arise post-deployment
- Backup of original view DDL taken prior to change
