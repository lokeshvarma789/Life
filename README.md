What to check first

Step 1 — Confirm which database you're actually connected to right now

sql
SELECT CURRENT SERVER FROM SYSIBM.SYSDUMMY1;

This works even on a minimal connection and tells you the actual database name AQT is talking to.

Step 2 — Confirm your current schema/user

sql
VALUES CURRENT SCHEMA;
VALUES CURRENT USER;

Step 3 — See what schemas actually exist in this connection

sql
SELECT DISTINCT TABSCHEMA
FROM SYSCAT.TABLES
ORDER BY TABSCHEMA;

If this last query itself fails (SYSCAT not found), that confirms SYSCAT genuinely isn't available in whatever you're connected to — meaning this connection is not the KCDWHPRC/KCINGPRC warehouse database at all.

What this likely means practically

Remember from your earlier sessions — you have two separate AQT connections:

PRODDM (or DUTDM) — the actual DB2 warehouse, where KCDWHPRC/KCINGPRC live
Possibly a different alias that only points to a smaller admin/system database

You may currently be querying the wrong connection alias in AQT — one that was set up for something else, or a default/blank one that never got the real DB2 catalog behind it.

What to do
Open your AQT connections list (the left panel with your saved connections)
Check which connection you're actually running queries against right now
Switch to (or reconnect to) your PRODDM or DUTDM connection — the ones Mario/Vinay were walked through setting up (alias LOCDB2Z, PRODDM, or similar naming)
Re-run:
sql
SELECT DISTINCT TABSCHEMA
FROM SYSCAT.TABLES
WHERE TABSCHEMA LIKE 'KC%'
ORDER BY TABSCHEMA;
