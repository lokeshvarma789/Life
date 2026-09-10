EXECUTE IMMEDIATE
$$
DECLARE

    V_SQL            STRING;
    V_SOURCE_TABLE   STRING;
    V_TARGET_TABLE   STRING;
    V_TARGET_SCHEMA  STRING;
    V_BK_SELECT      STRING;
    V_BK_JOIN        STRING;
    V_VERSION_COL    STRING;
    V_SOURCE_FROM    STRING;
    V_ERROR          STRING;

BEGIN

    /* =========================================================
       CLEAR OLD TEST RESULTS
       ========================================================= */

    DELETE FROM
        DEV_DATA_TELEMETRY.SRC_TGT_RECONCILIATION.RECON_RESULTS_TEST;


    /* =========================================================
       FRATDB ONLY

       1. SRC_BK_REF supplies Source Table + Business Key
       2. VAULTSPEED_METADATA_EXPORT supplies Raw Vault mapping
       3. Only tables existing in PROD_DV.LANDING_FRATDB
          are considered.
       ========================================================= */

    FOR REC IN
    (
        WITH BK_OBJECT AS
        (
            SELECT
                OBJECT_CONSTRUCT_KEEP_NULL(*) AS OBJ
            FROM
                DEV_DATA_TELEMETRY
                    .SRC_TGT_RECONCILIATION
                    .SRC_BK_REF
        ),

        BK_RAW AS
        (
            SELECT

                UPPER(
                    TRIM(
                        COALESCE(
                            OBJ:"SOURCE_TABLE"::STRING,
                            OBJ:"SRC_TABLE"::STRING,
                            OBJ:"TABLE_NAME"::STRING,
                            OBJ:"SOURCE_TABLE_NAME"::STRING,
                            OBJ:"SRC_TABLE_NAME"::STRING
                        )
                    )
                ) AS SOURCE_TABLE,

                COALESCE(
                    OBJ:"BUSINESS_KEYS"::STRING,
                    OBJ:"BUSINESS_KEY"::STRING,
                    OBJ:"SRC_BK"::STRING,
                    OBJ:"BK"::STRING,
                    OBJ:"BK_COLUMN"::STRING,
                    OBJ:"BUSINESS_KEY_COLUMN"::STRING
                ) AS BK_TEXT

            FROM BK_OBJECT
        ),

        /*
           Handles:
               ADDRESS_ID

           and composite BKs such as:
               CLIENT_ID, ADDRESS_ID
        */

        BK_SPLIT AS
        (
            SELECT DISTINCT

                B.SOURCE_TABLE,

                UPPER(
                    TRIM(F.VALUE::STRING)
                ) AS BK_COLUMN

            FROM BK_RAW B,

            LATERAL FLATTEN
            (
                INPUT =>
                    SPLIT(
                        REPLACE(B.BK_TEXT, ';', ','),
                        ','
                    )
            ) F

            WHERE
                    B.SOURCE_TABLE IS NOT NULL
                AND B.BK_TEXT IS NOT NULL
                AND TRIM(F.VALUE::STRING) <> ''
        ),

        /*
           Build the dynamic BK select list and
           source-target comparison condition.
        */

        BK_CONFIG AS
        (
            SELECT

                SOURCE_TABLE,

                LISTAGG(
                    '"' ||
                    REPLACE(BK_COLUMN,'"','""') ||
                    '"',
                    ', '
                )
                WITHIN GROUP
                (
                    ORDER BY BK_COLUMN
                ) AS BK_SELECT,

                LISTAGG(
                    'S."' ||
                    REPLACE(BK_COLUMN,'"','""') ||
                    '" IS NOT DISTINCT FROM T."' ||
                    REPLACE(BK_COLUMN,'"','""') ||
                    '"',
                    ' AND '
                )
                WITHIN GROUP
                (
                    ORDER BY BK_COLUMN
                ) AS BK_JOIN

            FROM BK_SPLIT

            GROUP BY SOURCE_TABLE
        ),

        /* =====================================================
           VAULTSPEED RAW VAULT MAPPING
           ===================================================== */

        META_OBJECT AS
        (
            SELECT
                OBJECT_CONSTRUCT_KEEP_NULL(*) AS OBJ
            FROM
                PROD_DV.METADATA.VAULTSPEED_METADATA_EXPORT
        ),

        META_RAW AS
        (
            SELECT

                UPPER(
                    TRIM(
                        COALESCE(
                            OBJ:"SOURCE_TABLE"::STRING,
                            OBJ:"SRC_TABLE"::STRING,
                            OBJ:"SOURCE_TABLE_NAME"::STRING,
                            OBJ:"SRC_TABLE_NAME"::STRING,
                            OBJ:"SOURCE_OBJECT"::STRING,
                            OBJ:"SOURCE_OBJECT_NAME"::STRING
                        )
                    )
                ) AS SOURCE_TABLE,

                UPPER(
                    TRIM(
                        COALESCE(
                            OBJ:"TARGET_TABLE"::STRING,
                            OBJ:"TGT_TABLE"::STRING,
                            OBJ:"TARGET_TABLE_NAME"::STRING,
                            OBJ:"TGT_TABLE_NAME"::STRING,
                            OBJ:"TARGET_OBJECT"::STRING,
                            OBJ:"TARGET_OBJECT_NAME"::STRING,
                            OBJ:"VAULT_TABLE"::STRING
                        )
                    )
                ) AS TARGET_TABLE

            FROM META_OBJECT
        ),

        /*
           Prefer VaultSpeed S_FRATDB objects because these
           represent the SAT/LDS/REF-style Raw Vault objects
           described in the reconciliation requirement.
        */

        META_MAP AS
        (
            SELECT
                SOURCE_TABLE,
                TARGET_TABLE

            FROM META_RAW

            WHERE
                    SOURCE_TABLE IS NOT NULL
                AND TARGET_TABLE IS NOT NULL

            QUALIFY
                ROW_NUMBER() OVER
                (
                    PARTITION BY SOURCE_TABLE

                    ORDER BY

                        CASE
                            WHEN TARGET_TABLE LIKE 'S_FRATDB%'
                                THEN 1

                            WHEN TARGET_TABLE LIKE '%FRATDB%'
                                THEN 2

                            ELSE 3
                        END,

                        TARGET_TABLE
                ) = 1
        ),

        /* =====================================================
           ONLY PHYSICAL FRATDB LANDING TABLES
           ===================================================== */

        FRATDB_TABLES AS
        (
            SELECT

                B.SOURCE_TABLE,
                B.BK_SELECT,
                B.BK_JOIN,

                M.TARGET_TABLE,

                TGT.TABLE_SCHEMA AS TARGET_SCHEMA,

                CASE
                    WHEN CT.TABLE_NAME IS NOT NULL
                        THEN TRUE
                    ELSE FALSE
                END AS HAS_CT,


                /* ---------------------------------------------
                   Pick historical-version column.

                   EFFECTIVE_DT has first priority.
                   --------------------------------------------- */

                CASE

                    WHEN EXISTS
                    (
                        SELECT 1
                        FROM PROD_DV.INFORMATION_SCHEMA.COLUMNS C

                        WHERE
                                C.TABLE_SCHEMA = 'LANDING_FRATDB'
                            AND C.TABLE_NAME =
                                    'TF_' || B.SOURCE_TABLE
                            AND UPPER(C.COLUMN_NAME) =
                                    'EFFECTIVE_DT'
                    )

                    AND EXISTS
                    (
                        SELECT 1
                        FROM PROD_DV.INFORMATION_SCHEMA.COLUMNS C

                        WHERE
                                C.TABLE_SCHEMA = TGT.TABLE_SCHEMA
                            AND C.TABLE_NAME = M.TARGET_TABLE
                            AND UPPER(C.COLUMN_NAME) =
                                    'EFFECTIVE_DT'
                    )

                    THEN 'EFFECTIVE_DT'


                    WHEN EXISTS
                    (
                        SELECT 1
                        FROM PROD_DV.INFORMATION_SCHEMA.COLUMNS C

                        WHERE
                                C.TABLE_SCHEMA = 'LANDING_FRATDB'
                            AND C.TABLE_NAME =
                                    'TF_' || B.SOURCE_TABLE
                            AND UPPER(C.COLUMN_NAME) =
                                    'PREVIOUS_UPDATE_TIMESTAMP'
                    )

                    AND EXISTS
                    (
                        SELECT 1
                        FROM PROD_DV.INFORMATION_SCHEMA.COLUMNS C

                        WHERE
                                C.TABLE_SCHEMA = TGT.TABLE_SCHEMA
                            AND C.TABLE_NAME = M.TARGET_TABLE
                            AND UPPER(C.COLUMN_NAME) =
                                    'PREVIOUS_UPDATE_TIMESTAMP'
                    )

                    THEN 'PREVIOUS_UPDATE_TIMESTAMP'


                    WHEN EXISTS
                    (
                        SELECT 1
                        FROM PROD_DV.INFORMATION_SCHEMA.COLUMNS C

                        WHERE
                                C.TABLE_SCHEMA = 'LANDING_FRATDB'
                            AND C.TABLE_NAME =
                                    'TF_' || B.SOURCE_TABLE
                            AND UPPER(C.COLUMN_NAME) =
                                    'UPDATE_TIMESTAMP'
                    )

                    AND EXISTS
                    (
                        SELECT 1
                        FROM PROD_DV.INFORMATION_SCHEMA.COLUMNS C

                        WHERE
                                C.TABLE_SCHEMA = TGT.TABLE_SCHEMA
                            AND C.TABLE_NAME = M.TARGET_TABLE
                            AND UPPER(C.COLUMN_NAME) =
                                    'UPDATE_TIMESTAMP'
                    )

                    THEN 'UPDATE_TIMESTAMP'


                    WHEN EXISTS
                    (
                        SELECT 1
                        FROM PROD_DV.INFORMATION_SCHEMA.COLUMNS C

                        WHERE
                                C.TABLE_SCHEMA = 'LANDING_FRATDB'
                            AND C.TABLE_NAME =
                                    'TF_' || B.SOURCE_TABLE
                            AND UPPER(C.COLUMN_NAME) =
                                    'LAST_UPDATE_TIMESTAMP'
                    )

                    AND EXISTS
                    (
                        SELECT 1
                        FROM PROD_DV.INFORMATION_SCHEMA.COLUMNS C

                        WHERE
                                C.TABLE_SCHEMA = TGT.TABLE_SCHEMA
                            AND C.TABLE_NAME = M.TARGET_TABLE
                            AND UPPER(C.COLUMN_NAME) =
                                    'LAST_UPDATE_TIMESTAMP'
                    )

                    THEN 'LAST_UPDATE_TIMESTAMP'

                    ELSE NULL

                END AS VERSION_COLUMN


            FROM BK_CONFIG B


            /* Source must physically exist in FRATDB */

            INNER JOIN
                PROD_DV.INFORMATION_SCHEMA.TABLES SRC

                ON  SRC.TABLE_SCHEMA = 'LANDING_FRATDB'
                AND SRC.TABLE_NAME =
                        'TF_' || B.SOURCE_TABLE


            /* __CT is optional */

            LEFT JOIN
                PROD_DV.INFORMATION_SCHEMA.TABLES CT

                ON  CT.TABLE_SCHEMA = 'LANDING_FRATDB'
                AND CT.TABLE_NAME =
                        'TF_' || B.SOURCE_TABLE || '__CT'


            LEFT JOIN META_MAP M

                ON M.SOURCE_TABLE = B.SOURCE_TABLE


            /*
              Resolve which PROD_DV schema actually contains
              the Raw Vault target.
            */

            LEFT JOIN
                PROD_DV.INFORMATION_SCHEMA.TABLES TGT

                ON TGT.TABLE_NAME = M.TARGET_TABLE


            QUALIFY
                ROW_NUMBER() OVER
                (
                    PARTITION BY B.SOURCE_TABLE

                    ORDER BY

                        CASE
                            WHEN TGT.TABLE_SCHEMA ILIKE '%VAULT%'
                                THEN 1
                            ELSE 2
                        END,

                        TGT.TABLE_SCHEMA
                ) = 1
        )


        SELECT *
        FROM FRATDB_TABLES

        ORDER BY SOURCE_TABLE
    )

    DO

        V_SOURCE_TABLE  := REC.SOURCE_TABLE;
        V_TARGET_TABLE  := REC.TARGET_TABLE;
        V_TARGET_SCHEMA := REC.TARGET_SCHEMA;
        V_BK_SELECT     := REC.BK_SELECT;
        V_BK_JOIN       := REC.BK_JOIN;
        V_VERSION_COL   := REC.VERSION_COLUMN;


        /* =====================================================
           NO TARGET MAPPING
           ===================================================== */

        IF (V_TARGET_TABLE IS NULL OR V_TARGET_SCHEMA IS NULL)
        THEN

            INSERT INTO
                DEV_DATA_TELEMETRY
                    .SRC_TGT_RECONCILIATION
                    .RECON_RESULTS_TEST
            (
                SOURCE_TABLE,
                TARGET_TABLE,
                SOURCE_COUNT,
                TARGET_COUNT,
                MISSING_RECORDS,
                STATUS,
                ERROR_MESSAGE
            )
            VALUES
            (
                :V_SOURCE_TABLE,
                NULL,
                NULL,
                NULL,
                NULL,
                'SKIPPED',
                'No Raw Vault SAT/LDS/REF mapping found in VAULTSPEED_METADATA_EXPORT'
            );


        /* =====================================================
           NO VALID VERSION COLUMN
           ===================================================== */

        ELSEIF (V_VERSION_COL IS NULL)
        THEN

            INSERT INTO
                DEV_DATA_TELEMETRY
                    .SRC_TGT_RECONCILIATION
                    .RECON_RESULTS_TEST
            (
                SOURCE_TABLE,
                TARGET_TABLE,
                SOURCE_COUNT,
                TARGET_COUNT,
                MISSING_RECORDS,
                STATUS,
                ERROR_MESSAGE
            )
            VALUES
            (
                :V_SOURCE_TABLE,
                :V_TARGET_TABLE,
                NULL,
                NULL,
                NULL,
                'SKIPPED',
                'No common historical version column found between Landing and Raw Vault'
            );


        ELSE

            /* =================================================
               BUILD SOURCE HISTORY

               BASE
                 +
               BASE__CT
               ================================================= */

            IF (REC.HAS_CT)
            THEN

                V_SOURCE_FROM :=

                    ' SELECT ' ||
                        V_BK_SELECT ||
                        ', "' || V_VERSION_COL ||
                        '" AS RECON_VERSION ' ||

                    ' FROM PROD_DV.LANDING_FRATDB."TF_' ||
                        REPLACE(V_SOURCE_TABLE,'"','""') ||
                        '" ' ||

                    ' UNION ALL ' ||

                    ' SELECT ' ||
                        V_BK_SELECT ||
                        ', "' || V_VERSION_COL ||
                        '" AS RECON_VERSION ' ||

                    ' FROM PROD_DV.LANDING_FRATDB."TF_' ||
                        REPLACE(V_SOURCE_TABLE,'"','""') ||
                        '__CT" ';

            ELSE

                V_SOURCE_FROM :=

                    ' SELECT ' ||
                        V_BK_SELECT ||
                        ', "' || V_VERSION_COL ||
                        '" AS RECON_VERSION ' ||

                    ' FROM PROD_DV.LANDING_FRATDB."TF_' ||
                        REPLACE(V_SOURCE_TABLE,'"','""') ||
                        '" ';

            END IF;


            /* =================================================
               RECONCILIATION

               SOURCE_COUNT:
                   DISTINCT BK + VERSION from Landing history

               TARGET_COUNT:
                   DISTINCT BK + VERSION in Raw Vault

               MISSING_RECORDS:
                   Landing BK + VERSION absent from Raw Vault

               PASS:
                   zero missing historical versions

               FAIL:
                   at least one historical version missing
               ================================================= */

            V_SQL :=

            ' INSERT INTO ' ||
            ' DEV_DATA_TELEMETRY.' ||
            ' SRC_TGT_RECONCILIATION.' ||
            ' RECON_RESULTS_TEST ' ||

            ' ( ' ||
            '   SOURCE_TABLE, ' ||
            '   TARGET_TABLE, ' ||
            '   SOURCE_COUNT, ' ||
            '   TARGET_COUNT, ' ||
            '   MISSING_RECORDS, ' ||
            '   STATUS, ' ||
            '   ERROR_MESSAGE ' ||
            ' ) ' ||


            ' WITH SOURCE_UNION AS ' ||
            ' ( ' ||

                    V_SOURCE_FROM ||

            ' ), ' ||


            ' SOURCE_HISTORY AS ' ||
            ' ( ' ||

            '     SELECT DISTINCT ' ||
                        V_BK_SELECT ||
            '          ,TRY_TO_TIMESTAMP_NTZ(' ||
            '               TO_VARCHAR(RECON_VERSION)' ||
            '           ) AS RECON_VERSION ' ||

            '     FROM SOURCE_UNION ' ||

            ' ), ' ||


            ' TARGET_HISTORY AS ' ||
            ' ( ' ||

            '     SELECT DISTINCT ' ||
                        V_BK_SELECT ||
            '          ,TRY_TO_TIMESTAMP_NTZ(' ||
            '               TO_VARCHAR("' ||
                            V_VERSION_COL ||
            '               ")' ||
            '           ) AS RECON_VERSION ' ||

            '     FROM PROD_DV."' ||
                    REPLACE(V_TARGET_SCHEMA,'"','""') ||
            '"."' ||
                    REPLACE(V_TARGET_TABLE,'"','""') ||
            '" ' ||

            ' ), ' ||


            ' MISSING_HISTORY AS ' ||
            ' ( ' ||

            '     SELECT S.* ' ||

            '     FROM SOURCE_HISTORY S ' ||

            '     WHERE NOT EXISTS ' ||
            '     ( ' ||

            '         SELECT 1 ' ||
            '         FROM TARGET_HISTORY T ' ||

            '         WHERE ' ||
                        V_BK_JOIN ||

            '           AND S.RECON_VERSION ' ||
            '               IS NOT DISTINCT FROM ' ||
            '               T.RECON_VERSION ' ||

            '     ) ' ||

            ' ), ' ||


            ' COUNTS AS ' ||
            ' ( ' ||

            '     SELECT ' ||

            '       (SELECT COUNT(*) ' ||
            '          FROM SOURCE_HISTORY) ' ||
            '              AS SOURCE_COUNT, ' ||

            '       (SELECT COUNT(*) ' ||
            '          FROM TARGET_HISTORY) ' ||
            '              AS TARGET_COUNT, ' ||

            '       (SELECT COUNT(*) ' ||
            '          FROM MISSING_HISTORY) ' ||
            '              AS MISSING_RECORDS ' ||

            ' ) ' ||


            ' SELECT ' ||

            '''' ||
                REPLACE(V_SOURCE_TABLE,'''','''''') ||
            ''',' ||

            '''' ||
                REPLACE(V_TARGET_TABLE,'''','''''') ||
            ''',' ||

            ' SOURCE_COUNT, ' ||
            ' TARGET_COUNT, ' ||
            ' MISSING_RECORDS, ' ||

            ' CASE ' ||
            '   WHEN MISSING_RECORDS = 0 ' ||
            '       THEN ''PASS'' ' ||
            '   ELSE ''FAIL'' ' ||
            ' END, ' ||

            ' NULL ' ||

            ' FROM COUNTS ';


            /* =================================================
               RUN THIS TABLE
               ================================================= */

            BEGIN

                EXECUTE IMMEDIATE :V_SQL;

            EXCEPTION

                WHEN OTHER THEN

                    V_ERROR := SQLERRM;

                    INSERT INTO
                        DEV_DATA_TELEMETRY
                            .SRC_TGT_RECONCILIATION
                            .RECON_RESULTS_TEST
                    (
                        SOURCE_TABLE,
                        TARGET_TABLE,
                        SOURCE_COUNT,
                        TARGET_COUNT,
                        MISSING_RECORDS,
                        STATUS,
                        ERROR_MESSAGE
                    )
                    VALUES
                    (
                        :V_SOURCE_TABLE,
                        :V_TARGET_TABLE,
                        NULL,
                        NULL,
                        NULL,
                        'ERROR',
                        :V_ERROR
                    );

            END;

        END IF;

    END FOR;

END;
$$;
