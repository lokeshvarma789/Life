-- Q7 — Daily Volume + Frequency Validation (Siddesh Step 5)
-- One row per CT table per operation (INSERT/UPDATE/DELETE).
-- Compares today against the table's own 30-day history for that operation.
-- SD_VERDICT, IQR_VERDICT, and FREQUENCY_VERDICT are independent columns.

EXECUTE IMMEDIATE
$$
DECLARE
    sql_stmt STRING;
    rs RESULTSET;
BEGIN

    SELECT LISTAGG(
        'SELECT ''' || TABLE_SCHEMA || ''' AS SCHEMA_NAME, ''' ||
        TABLE_NAME || ''' AS TABLE_NAME, ' ||
        'OPERATION, ' ||
        'TODAY_COUNT, ' ||
        'ROUND(HIST_30D_AVG, 1)    AS HIST_30D_AVG, ' ||
        'ROUND(HIST_30D_STDDEV, 1) AS HIST_30D_STDDEV, ' ||
        'ROUND(HIST_30D_25PCT, 1)  AS HIST_30D_25PCT, ' ||
        'ROUND(HIST_30D_75PCT, 1)  AS HIST_30D_75PCT, ' ||
        'HIST_30D_DAYS_WITH_LOAD, ' ||
        'CASE ' ||
        '  WHEN HIST_30D_DAYS_WITH_LOAD >= 25 THEN ''DAILY'' ' ||
        '  WHEN HIST_30D_DAYS_WITH_LOAD BETWEEN 15 AND 24 THEN ''NEAR_DAILY'' ' ||
        '  WHEN ACTIVE_DAYS_60 BETWEEN 6 AND 14 THEN ''WEEKLY'' ' ||
        '  WHEN ACTIVE_DAYS_90 BETWEEN 2 AND 5 THEN ''MONTHLY'' ' ||
        '  ELSE ''IRREGULAR'' ' ||
        'END AS LOAD_PATTERN, ' ||
        'CASE ' ||
        '  WHEN HIST_30D_DAYS_WITH_LOAD < 5 THEN ''LOW_CONFIDENCE'' ' ||
        '  WHEN TODAY_COUNT > HIST_30D_AVG + 3 * HIST_30D_STDDEV THEN ''CRITICAL - HIGH (3 sigma)'' ' ||
        '  WHEN TODAY_COUNT < HIST_30D_AVG - 3 * HIST_30D_STDDEV THEN ''CRITICAL - LOW (3 sigma)'' ' ||
        '  WHEN TODAY_COUNT > HIST_30D_AVG + 2 * HIST_30D_STDDEV THEN ''WARNING - HIGH (2 sigma)'' ' ||
        '  WHEN TODAY_COUNT < HIST_30D_AVG - 2 * HIST_30D_STDDEV THEN ''WARNING - LOW (2 sigma)'' ' ||
        '  ELSE ''NORMAL'' ' ||
        'END AS SD_VERDICT, ' ||
        'CASE ' ||
        '  WHEN HIST_30D_DAYS_WITH_LOAD < 5 THEN ''LOW_CONFIDENCE'' ' ||
        '  WHEN TODAY_COUNT > HIST_30D_75PCT + 1.5 * (HIST_30D_75PCT - HIST_30D_25PCT) THEN ''HIGH OUTLIER (IQR)'' ' ||
        '  WHEN TODAY_COUNT < HIST_30D_25PCT - 1.5 * (HIST_30D_75PCT - HIST_30D_25PCT) THEN ''LOW OUTLIER (IQR)'' ' ||
        '  ELSE ''NORMAL'' ' ||
        'END AS IQR_VERDICT, ' ||
        'CASE ' ||
        '  WHEN HIST_30D_DAYS_WITH_LOAD >= 25 AND TODAY_COUNT = 0 THEN ''MISS - DAILY pattern, no load today'' ' ||
        '  WHEN HIST_30D_DAYS_WITH_LOAD BETWEEN 15 AND 24 AND TODAY_COUNT = 0 AND DAYS_SINCE_LAST > 2 THEN ''MISS - NEAR_DAILY overdue'' ' ||
        '  WHEN ACTIVE_DAYS_60 BETWEEN 6 AND 14 AND DAYS_SINCE_LAST > 10 THEN ''MISS - WEEKLY overdue'' ' ||
        '  WHEN ACTIVE_DAYS_90 BETWEEN 2 AND 5 AND DAYS_SINCE_LAST > 40 THEN ''MISS - MONTHLY overdue'' ' ||
        '  WHEN ACTIVE_DAYS_90 = 1 AND TODAY_COUNT > 0 THEN ''UNEXPECTED LOAD - IRREGULAR pattern usually quiet'' ' ||
        '  ELSE ''ON SCHEDULE'' ' ||
        'END AS FREQUENCY_VERDICT ' ||
        'FROM ( ' ||
            'SELECT ' ||
            '  OPERATION, ' ||
            '  SUM(CASE WHEN LOAD_DATE = CURRENT_DATE() THEN DAILY_COUNT ELSE 0 END) AS TODAY_COUNT, ' ||
            '  AVG(CASE WHEN LOAD_DATE < CURRENT_DATE() THEN DAILY_COUNT END)    AS HIST_30D_AVG, ' ||
            '  STDDEV(CASE WHEN LOAD_DATE < CURRENT_DATE() THEN DAILY_COUNT END) AS HIST_30D_STDDEV, ' ||
            '  PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY CASE WHEN LOAD_DATE < CURRENT_DATE() THEN DAILY_COUNT END) AS HIST_30D_25PCT, ' ||
            '  PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY CASE WHEN LOAD_DATE < CURRENT_DATE() THEN DAILY_COUNT END) AS HIST_30D_75PCT, ' ||
            '  COUNT(DISTINCT CASE WHEN LOAD_DATE < CURRENT_DATE() THEN LOAD_DATE END) AS HIST_30D_DAYS_WITH_LOAD, ' ||
            '  MAX(ACTIVE_DAYS_60) AS ACTIVE_DAYS_60, ' ||
            '  MAX(ACTIVE_DAYS_90) AS ACTIVE_DAYS_90, ' ||
            '  MAX(DAYS_SINCE_LAST) AS DAYS_SINCE_LAST ' ||
            'FROM ( ' ||
                'SELECT HEADER__OPERATION AS OPERATION, ' ||
                '       TO_DATE(HEADER__TIMESTAMP) AS LOAD_DATE, ' ||
                '       COUNT(*) AS DAILY_COUNT, ' ||
                '       (SELECT COUNT(DISTINCT TO_DATE(HEADER__TIMESTAMP)) ' ||
                '        FROM PROD_LANDING.' || TABLE_SCHEMA || '.' || TABLE_NAME ||
                '        WHERE HEADER__TIMESTAMP >= DATEADD(DAY, -60, CURRENT_DATE())) AS ACTIVE_DAYS_60, ' ||
                '       (SELECT COUNT(DISTINCT TO_DATE(HEADER__TIMESTAMP)) ' ||
                '        FROM PROD_LANDING.' || TABLE_SCHEMA || '.' || TABLE_NAME ||
                '        WHERE HEADER__TIMESTAMP >= DATEADD(DAY, -90, CURRENT_DATE())) AS ACTIVE_DAYS_90, ' ||
                '       (SELECT DATEDIFF(DAY, MAX(TO_DATE(HEADER__TIMESTAMP)), CURRENT_DATE()) ' ||
                '        FROM PROD_LANDING.' || TABLE_SCHEMA || '.' || TABLE_NAME ||
                '        WHERE HEADER__TIMESTAMP >= DATEADD(DAY, -90, CURRENT_DATE())) AS DAYS_SINCE_LAST ' ||
                'FROM PROD_LANDING.' || TABLE_SCHEMA || '.' || TABLE_NAME || ' ' ||
                'WHERE HEADER__TIMESTAMP >= DATEADD(DAY, -30, CURRENT_DATE()) ' ||
                '  AND HEADER__OPERATION IN (''INSERT'', ''UPDATE'', ''DELETE'') ' ||
                'GROUP BY 1, 2 ' ||
            ') DAILY_HIST ' ||
            'GROUP BY OPERATION ' ||
        ') X',
        ' UNION ALL '
    )
    INTO :sql_stmt
    FROM PROD_LANDING.INFORMATION_SCHEMA.TABLES
    WHERE TABLE_TYPE = 'BASE TABLE'
      AND TABLE_SCHEMA IN ('ING','FRATDB','RPLUS','FRAT_ING','DI','DMC','L70','DCLM','SEI','SF','SAP','LTC')
      AND TABLE_NAME LIKE '%__CT';

    sql_stmt := 'SELECT * FROM (' || :sql_stmt || ') ' ||
                'WHERE SD_VERDICT NOT IN (''NORMAL'', ''LOW_CONFIDENCE'') ' ||
                '   OR IQR_VERDICT NOT IN (''NORMAL'', ''LOW_CONFIDENCE'') ' ||
                '   OR FREQUENCY_VERDICT <> ''ON SCHEDULE'' ' ||
                'ORDER BY SCHEMA_NAME, TABLE_NAME, OPERATION';

    rs := (EXECUTE IMMEDIATE :sql_stmt);
    RETURN TABLE(rs);

END;
$$;
