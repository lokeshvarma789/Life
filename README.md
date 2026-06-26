Layer 1: Syntax Check
Just run the query as-is on ING.TPOL__CT. If it returns 1 row with all 30+ columns populated and no errors — Layer 1 passes.
If it errors, paste the error and I'll fix it.

Layer 2: Logic Spot-Check (Most Important)
Run these companion queries side-by-side with Step 5 and manually compare the numbers.
Validation Query A — Verify TODAY'S counts
sql-- Should match INSERTS_TODAY, UPDATES_TODAY, DELETES_TODAY, TOTAL_TODAY
SELECT
    SUM(CASE WHEN HEADER__OPERATION = 'INSERT' THEN 1 ELSE 0 END) AS INSERTS,
    SUM(CASE WHEN HEADER__OPERATION = 'UPDATE' THEN 1 ELSE 0 END) AS UPDATES,
    SUM(CASE WHEN HEADER__OPERATION = 'DELETE' THEN 1 ELSE 0 END) AS DELETES,
    COUNT(*)                                                       AS TOTAL
FROM PROD_LANDING.ING.TPOL__CT
WHERE DATE(HEADER__TIMESTAMP) = CURRENT_DATE();
✅ Numbers must match Step 5's "TODAY'S LOAD" section.
Validation Query B — Verify 60-DAY BASELINE numbers
sql-- Should match HIST_AVG_TOTAL, HIST_STDDEV_TOTAL, HIST_MIN_TOTAL, HIST_MAX_TOTAL
WITH daily_totals AS (
    SELECT DATE(HEADER__TIMESTAMP) AS LOAD_DT, COUNT(*) AS TOT_CNT
    FROM PROD_LANDING.ING.TPOL__CT
    WHERE HEADER__TIMESTAMP >= DATEADD(DAY, -60, CURRENT_DATE())
      AND DATE(HEADER__TIMESTAMP) < CURRENT_DATE()
    GROUP BY DATE(HEADER__TIMESTAMP)
)
SELECT
    COUNT(*)                                                   AS DAYS_LOADED,
    ROUND(AVG(TOT_CNT), 0)                                     AS AVG_TOTAL,
    ROUND(STDDEV(TOT_CNT), 0)                                  AS STDDEV_TOTAL,
    MIN(TOT_CNT)                                               AS MIN_TOTAL,
    MAX(TOT_CNT)                                               AS MAX_TOTAL,
    ROUND(PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY TOT_CNT), 0) AS Q1,
    ROUND(PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY TOT_CNT), 0) AS MEDIAN,
    ROUND(PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY TOT_CNT), 0) AS Q3
FROM daily_totals;
✅ All 8 numbers must match Step 5's "HISTORICAL BASELINE" section exactly.
Validation Query C — Verify STDDEV bounds calculation
Manual math:
STDDEV_LOWER_BOUND = MAX(0, HIST_AVG - 2 * HIST_STDDEV)
STDDEV_UPPER_BOUND = HIST_AVG + 2 * HIST_STDDEV
Pick up the AVG and STDDEV from Validation B, calculate manually, compare to Step 5 output. They must match.
Validation Query D — Verify IQR bounds calculation
Manual math:
IQR = Q3 - Q1
IQR_LOWER_BOUND = MAX(0, Q1 - 1.5 * IQR)
IQR_UPPER_BOUND = Q3 + 1.5 * IQR
Validation Query E — Verify frequency / last load date
sqlSELECT
    MAX(DATE(HEADER__TIMESTAMP))                              AS LAST_LOAD_DT,
    DATEDIFF(DAY, MAX(DATE(HEADER__TIMESTAMP)), CURRENT_DATE()) AS DAYS_SINCE_LOAD
FROM PROD_LANDING.ING.TPOL__CT;
