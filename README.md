
SELECT
    COLUMN_NAME,
    DATA_TYPE,
    ORDINAL_POSITION
FROM PROD_LANDING.INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'ING'
  AND TABLE_NAME = 'TCVGA__CT'
ORDER BY ORDINAL_POSITION;

Look for two categories of columns:

Qlik/CDC metadata columns
Original business columns from TCVGA

Possible metadata columns may contain terms such as:

HEADER
CHANGE
OPERATION
TIMESTAMP
POSITION

Record the exact names you see; do not guess their meanings yet.

Step 3 — Find which account writes to TCVGA__CT

The strongest Snowflake evidence is ACCESS_HISTORY, because it can identify objects modified by write operations. Snowflake records write-related object information in OBJECTS_MODIFIED; availability depends on your Snowflake edition and permissions.

Try:

SELECT
    AH.QUERY_START_TIME,
    AH.USER_NAME,
    AH.QUERY_ID,
    AH.OBJECTS_MODIFIED
FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY AH
WHERE AH.QUERY_START_TIME >= DATEADD(DAY, -30, CURRENT_TIMESTAMP())
  AND TO_VARCHAR(AH.OBJECTS_MODIFIED)
      ILIKE '%PROD_LANDING.ING.TCVGA__CT%'
ORDER BY AH.QUERY_START_TIME DESC;

This should help identify:

Who wrote to the table
When it was written
The query ID
The modified object

Then use the query ID:

SELECT
    START_TIME,
    USER_NAME,
    ROLE_NAME,
    WAREHOUSE_NAME,
    QUERY_TAG,
    QUERY_TYPE,
    QUERY_TEXT,
    EXECUTION_STATUS
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE QUERY_ID = 'PASTE_QUERY_ID_HERE';

Snowflake QUERY_HISTORY contains details such as the user, role, warehouse, query type, query tag, SQL text, and execution status.

What we are looking for

The result may show something like:

USER_NAME      = QLIK_SERVICE_ACCOUNT
WAREHOUSE_NAME = QLIK_WH
QUERY_TAG       = ...
QUERY_TYPE      = COPY / INSERT / MERGE

The actual names in your environment may be different. We will use the returned values as evidence.

If you cannot access ACCESS_HISTORY

Run the simpler query-history search:

SELECT
    START_TIME,
    USER_NAME,
    ROLE_NAME,
    WAREHOUSE_NAME,
    QUERY_TAG,
    QUERY_TYPE,
    QUERY_TEXT,
    EXECUTION_STATUS
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE START_TIME >= DATEADD(DAY, -30, CURRENT_TIMESTAMP())
  AND QUERY_TEXT ILIKE '%TCVGA__CT%'
ORDER BY START_TIME DESC;

If that gives no result, broaden it:

SELECT
    START_TIME,
    USER_NAME,
    ROLE_NAME,
    WAREHOUSE_NAME,
    QUERY_TAG,
    QUERY_TYPE,
    QUERY_TEXT,
    EXECUTION_STATUS
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE START_TIME >= DATEADD(DAY, -30, CURRENT_TIMESTAMP())
  AND QUERY_TEXT ILIKE '%TCVGA%'
ORDER BY START_TIME DESC;

ACCOUNT_USAGE.QUERY_HISTORY can provide historical query activity, although access depends on the privileges assigned to your role.

Step 4 — Use the result to find the Qlik task

After Snowflake shows that a Qlik-related account is loading the table, go into the Qlik Replicate or Qlik Data Integration interface.

Search using:

TCVGA__CT
TCVGA
ING
PROD_LANDING

Open the matching replication task.

Inside that task, capture these exact fields:

Information	Where to inspect
Task name	Task header/details
Source endpoint	Task source connection
Target endpoint	Task target connection
Source schema	Selected tables/table mapping
Source table	Selected tables/table mapping
Target database	Target settings
Target schema	Target settings or mapping
Target table	Table mapping
Load method	Full Load / Store Changes / CDC
Change-table suffix	Store Changes settings

We are specifically trying to discover:

Qlik Task Name
Source Endpoint Name
Source Database Type
Source Server/Location
Source Schema
Source Table
Target Schema
Target Table
The important Qlik screen

Find the section showing the source-to-target table mapping.

It should reveal something similar to:

Source endpoint: ING_PROD_DB2
Source schema:   KCINGPRC
Source table:    TCVGA

Target endpoint: SNOWFLAKE_PROD
Target database: PROD_LANDING
Target schema:   ING
Target table:    TCVGA
Change table:    TCVGA__CT

That is where we truly discover KCINGPRC.

We must not conclude that KCINGPRC.TCVGA is the source only from the Snowflake table name, because Qlik can apply renaming and transformation rules to change-table names or mappings.

Step 5 — Verify the discovered source in AQT

Only after Qlik tells us the source endpoint and schema do we open AQT.

Suppose Qlik shows:

Server/location: LOCDB2Z
Schema: KCINGPRC
Table: TCVGA

Connect to LOCDB2Z and run:

SELECT
    CREATOR,
    NAME,
    TYPE
FROM SYSIBM.SYSTABLES
WHERE CREATOR = 'KCINGPRC'
  AND NAME = 'TCVGA';

Then verify columns:

SELECT
    TBCREATOR,
    TBNAME,
    NAME AS COLUMN_NAME,
    COLNO,
    COLTYPE,
    LENGTH,
    SCALE,
    NULLS
FROM SYSIBM.SYSCOLUMNS
WHERE TBCREATOR = 'KCINGPRC'
  AND TBNAME = 'TCVGA'
ORDER BY COLNO;

Now AQT confirms:

Server: LOCDB2Z
Schema: KCINGPRC
Table: TCVGA
Columns: verified
The evidence chain we are building

For every _CT table, create one row like this:

Field	Value	Evidence
Snowflake change table	PROD_LANDING.ING.TCVGA__CT	Snowflake
Related target/base table	Pending verification	Snowflake/Qlik
Loader username	Pending	Snowflake history
Qlik task name	Pending	Qlik
Qlik source endpoint	Pending	Qlik
Source technology	Pending	Qlik endpoint
Source server	Pending	Qlik endpoint
Source schema	Pending	Qlik table mapping
Source table	Pending	Qlik table mapping
Source verification	Pending	AQT
Load type	Pending	Qlik task settings
Downstream Talend job	Later	Talend
Correct investigation direction
1. PROD_LANDING.ING.TCVGA__CT
2. Identify who writes it
3. Find the Qlik replication task
4. Open the Qlik source endpoint
5. Find the DB2 server/location
6. Open the selected-table mapping
7. Find source schema and source table
8. Verify the source using AQT
9. Then trace forward from _CT into Talend and staging

For now, run only these two commands:

SHOW TABLES LIKE 'TCVGA%' IN SCHEMA PROD_LANDING.ING;
DESC TABLE PROD_LANDING.ING.TCVGA__CT;
