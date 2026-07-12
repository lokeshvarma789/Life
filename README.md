-- Does the historical row's DIV_EFF_YR representation match cleanly
-- against a straightforward YEAR(DIV_EFF_DT) computation, or is there
-- a stored value that's drifted from the derived value?
SELECT
    L70PUA_FCT_ID, CO_ID, POL_ID, CVG_NUM, L7FD_PECD,
    DIV_EFF_DT,
    DIV_EFF_YR AS STORED_DIV_EFF_YR,
    YEAR(DIV_EFF_DT) AS COMPUTED_DIV_EFF_YR,
    RPT_PRCES_STATS_ID,
    CRNT_REC_IND
FROM KCDWHPRC.T8LPUA
WHERE CO_ID = '01' AND POL_ID = '01120182' AND CVG_NUM = '01' AND L7FD_PECD = '0'
ORDER BY RPT_PRCES_STATS_ID;
