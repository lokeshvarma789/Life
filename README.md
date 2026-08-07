Step 1 — Open your query tool

Go to Snowflake (the same tool where you ran your earlier queries — the one with the blue Play button).

Step 2 — Take your "before" photo (run this first, save the results)

Copy this exactly, paste it in a new query tab, and run it:

sql
SELECT MB_NMBR, MEMBER_TYPE_CODE, MEMBER_TYPE_DESC, MEMBER_CLASS_CODE, MEMBER_CLASS_DESC
FROM INFO_MART.D_MEMBER
WHERE MB_NMBR IN (1378227, 621954, 3480288, 496166, 1756448, 4109793, 1889587)
ORDER BY MB_NMBR;

What to do with the result: take a screenshot of it. This is your "before" picture. Don't delete this screenshot — you'll need it later to compare.

Step 3 — Count how many total rows exist right now (before photo #2)
sql
SELECT COUNT(*) AS total_rows
FROM INFO_MART.D_MEMBER;

What to do: you'll get back one number. Write it down somewhere — a notepad, a sticky note, doesn't matter. Example: "Before fix: 45,231 rows."

Step 4 — Check there are no duplicate members right now
sql
SELECT MB_NMBR, COUNT(*) AS record_count
FROM INFO_MART.D_MEMBER
GROUP BY MB_NMBR
HAVING COUNT(*) > 1;

What you're expecting: this should show nothing — zero rows returned. That's good, that's what you want.

If it does show rows: stop, take a screenshot, and tell me — that means there's already a duplicate problem separate from your fix, and we need to look at that first.

Step 5 — Now go make the actual fix

This step happens in Notepad++, where you already have the .fex/view code open (from the screenshots you showed me earlier).

You're looking for these two lines in BR_STD_MEMBER:

Find this:

sql
BUSINESS_RULES.UDF_LOOKUP_EDIT_VALUE('MB-TYPE', SM.MB_TYPE) AS Member_Class_Desc

Change it to:

sql
BUSINESS_RULES.UDF_LOOKUP_EDIT_VALUE('MB-CLASS', SM.MB_CLASS) AS Member_Class_Desc

And find this:

sql
BUSINESS_RULES.UDF_LOOKUP_EDIT_VALUE('MB-CLASS', SM.MB_CLASS) AS Member_Type_Desc

Change it to:

sql
BUSINESS_RULES.UDF_LOOKUP_EDIT_VALUE('MB-TYPE', SM.MB_TYPE) AS Member_Type_Desc

That's it. Only these two lines change. Nothing else in the whole file gets touched.

Important: don't do this in Prod. Do this in Dev only first. If you're not sure how to actually push this change so the database sees it (this might need a DBA to run a "recreate view" command), stop here and tell me — we'll figure that part out together before going further.

Step 6 — After the fix is live in Dev, take your "after" photo

Run the exact same query from Step 2 again:

sql
SELECT MB_NMBR, MEMBER_TYPE_CODE, MEMBER_TYPE_DESC, MEMBER_CLASS_CODE, MEMBER_CLASS_DESC
FROM INFO_MART.D_MEMBER
WHERE MB_NMBR IN (1378227, 621954, 3480288, 496166, 1756448, 4109793, 1889587)
ORDER BY MB_NMBR;

Take a screenshot of this result too.

Now compare your two screenshots side by side (before vs after).

What you're looking for: for the row where MB_NMBR = 1378227, the MEMBER_TYPE_CODE was D. Before the fix, MEMBER_TYPE_DESC showed "Regular" (wrong). After the fix, it should now show something like "Deceased" (correct). The values should have swapped.

Step 7 — Run your row count again — must match Step 3
sql
SELECT COUNT(*) AS total_rows
FROM INFO_MART.D_MEMBER;

Compare this number to what you wrote down in Step 3. These two numbers MUST be exactly the same.

If they match → good, the fix didn't accidentally add or remove any records
If they don't match → something went wrong, stop and tell me the two numbers
Step 8 — Run your duplicate check again — must still be empty
sql
SELECT MB_NMBR, COUNT(*) AS record_count
FROM INFO_MART.D_MEMBER
GROUP BY MB_NMBR
HAVING COUNT(*) > 1;

This must still show zero rows, exactly like Step 4. If it now shows duplicates that weren't there before, stop and tell me.

Step 9 — Send your before/after screenshots to Ryan Lavelle

Ryan is the person who reported this bug. Send him:

Your "before" screenshot (Step 2)
Your "after" screenshot (Step 6)

Ask him plainly: "Does this look correct now?" Wait for his yes before moving forward.
