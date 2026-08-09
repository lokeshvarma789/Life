The one and only code change

Find this in AF110.fex around line 210 — DEFINE FILE AF110_NAIFC:

Replace this old block:

PROD_AWARD/A10 = IF (PROD_AWARD_AMT GE 0) AND (PROD_AWARD_AMT LE 29999) THEN 'None' ELSE
                 IF (PROD_AWARD_AMT GE 30000) AND (PROD_AWARD_AMT LE 39999) THEN 'Bronze' ELSE
                 IF (PROD_AWARD_AMT GE 40000) AND (PROD_AWARD_AMT LE 49999) THEN 'Silver' ELSE
                 IF (PROD_AWARD_AMT GE 50000) AND (PROD_AWARD_AMT LE 64999) THEN 'Gold' ELSE
                 IF (PROD_AWARD_AMT GE 65000) AND (PROD_AWARD_AMT LE 89999) THEN 'Platinum' ELSE
                 IF (PROD_AWARD_AMT GE 90000) AND (PROD_AWARD_AMT LE 124999) THEN 'President' ELSE
                 IF (PROD_AWARD_AMT GE 125000) THEN 'Chairman' ELSE ' ';

With this new block:

-* RITM0134166 - Updated Production Award categories per Kerri McCarty / Alyssa Carrano 08-07-2026
-* Removed Bronze and Silver tiers. None = less than $75,000.
-* Gold=$75K-$99,999. Platinum=$100K-$149,999. President=$150K-$199,999. Chairman=$200K+.
PROD_AWARD/A10 = IF (PROD_AWARD_AMT GE 0) AND (PROD_AWARD_AMT LE 74999) THEN 'None' ELSE
                 IF (PROD_AWARD_AMT GE 75000) AND (PROD_AWARD_AMT LE 99999) THEN 'Gold' ELSE
                 IF (PROD_AWARD_AMT GE 100000) AND (PROD_AWARD_AMT LE 149999) THEN 'Platinum' ELSE
                 IF (PROD_AWARD_AMT GE 150000) AND (PROD_AWARD_AMT LE 199999) THEN 'President' ELSE
                 IF (PROD_AWARD_AMT GE 200000) THEN 'Chairman' ELSE ' ';

Also add to the Maintenance Log at the top of the file:

-* 08-07-2026  LV   RITM0134166 Updated Production Award categories.
-*                  Removed Bronze and Silver. None=<$75K. Gold=$75K-$99K.
-*                  Platinum=$100K-$149K. President=$150K-$199K. Chairman=$200K+.
-*                  FIC logic unchanged per business (Kerri McCarty 08-07-2026).
After making the change — test checklist

Temporarily uncomment and set the test date:

-SET &OVRDSYSDT = '20251231';

Run the report and check the Excel output. Confirm:
