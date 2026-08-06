Query 1 — Check what you have DIRECT access to
sql
SELECT TABSCHEMA, TABNAME, PRIVILEGE
FROM SYSCAT.TABAUTH
WHERE GRANTEE = CURRENT_USER
  AND TABSCHEMA = 'KCINGPRC'
ORDER BY TABNAME;

This shows every table where you personally were granted a permission (SELECT, INSERT, etc.) in the KCINGPRC schema.

Query 2 — Check access granted through a GROUP (very common in enterprise DB2)

Most of the time, access isn't granted to your user ID directly — it's granted to a group/role you belong to. This is likely why Query 1 might come back mostly empty even though you clearly have some access.

sql
SELECT TABSCHEMA, TABNAME, PRIVILEGE, GRANTEE, GRANTEETYPE
FROM SYSCAT.TABAUTH
WHERE TABSCHEMA = 'KCINGPRC'
  AND GRANTEE IN (
      SELECT GRANTEE FROM SYSCAT.DBAUTH WHERE GRANTEE = CURRENT_USER
      UNION
      SELECT CURRENT_USER FROM SYSIBM.SYSDUMMY1
  )
ORDER BY TABNAME;

If this feels too complex to get right on the first try (group membership queries vary by DB2 setup), there's a much simpler and more reliable approach below.

Query 3 — The simplest, most reliable method (recommended)

DB2 has a built-in function that tells you directly whether you can access an object — no guessing about groups needed:

sql
SELECT TABNAME,
       AUTH_CHECK_TABLE_USE(TRIM(TABSCHEMA) || '.' || TRIM(TABNAME), 'SELECT') AS CAN_SELECT
FROM SYSCAT.TABLES
WHERE TABSCHEMA = 'KCINGPRC'
ORDER BY TABNAME;
CAN_SELECT = 1 → you have access, safe to include in your comparison
CAN_SELECT = 0 → you don't have access, skip it, this is exactly why you're getting errors on it

This is the one I'd actually use. It checks your real, effective access (including group-based grants) in one shot, against every table in the schema, no trial-and-error.

How to use this to build your comparison list

Once you run Query 3, filter to just the accessible ones:

sql
SELECT TABNAME
FROM (
    SELECT TABNAME,
           AUTH_CHECK_TABLE_USE(TRIM(TABSCHEMA) || '.' || TRIM(TABNAME), 'SELECT') AS CAN_SELECT
    FROM SYSCAT.TABLES
    WHERE TABSCHEMA = 'KCINGPRC'
) 
WHERE CAN_SELECT = 1
ORDER BY TABNAME;

This gives you a clean list — just the table names you're actually allowed to query. Cross-reference this against your Snowflake source list, and only run the comparison for tables that appear in both lists. Anything in Snowflake but missing from this accessible list = you need to request access before you can validate that one; don't waste time troubleshooting an error that's actually just a permissions gap.
