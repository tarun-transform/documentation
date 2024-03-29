# Pricing Test Dashboard

## Readable query

```sql
WITH creates AS (
    SELECT 
        mtm.TEST_NAME,
        mtm.TEST_CATEGORY,
        mtm.TREATMENT,
        mtm.ZIP_CODE,
        mtm.PRICING_MARKET,
        mtm.TEST_CELL,
        mtm.TEST_START,
        mtm.TEST_END,
        mtm.TEST_DESCRIPTION,
        CAST(FLOOR((sop.SO_CRT_DT - mtm.TEST_START) / 7) AS INTEGER) AS WEEK_FROM_TEST_START,
        COUNT(DISTINCT sop.SO_NO || sop.SVC_UN_NO) AS CREATES,
        COUNT(DISTINCT CASE 
            WHEN sop.SO_STS_DESC = 'CA - Cancelled' AND sop.CYCLE_TIME <= 7 THEN sop.SO_NO || sop.SVC_UN_NO 
            END) AS CANCELS_7DAY
    FROM 
        FINANCE_ANALYTICS.ADHOC_TBLS.MARKET_TESTING_METADATA mtm
        INNER JOIN FINANCE_ANALYTICS.ADHOC_TBLS.UE_SERVICE_ORDER_PROFIT sop 
        ON mtm.ZIP_CODE = sop.ZIP_CODE
        AND sop.SO_CRT_DT BETWEEN mtm.TEST_START - 35 AND mtm.TEST_END
        AND sop.CALL_TYPE = 'D2C'
        AND (mtm.PRODUCT_CAT IS NULL OR UPPER(mtm.PRODUCT_CAT) = UPPER(sop.PRODCAT))
    GROUP BY 
        1, 2, 3, 4, 5, 6, 7, 8, 9, 10
),
sns_data AS (
    SELECT 
        mtm.TEST_NAME,
        mtm.TEST_CATEGORY,
        mtm.TREATMENT,
        mtm.ZIP_CODE,
        mtm.PRICING_MARKET,
        mtm.TEST_CELL,
        mtm.TEST_START,
        mtm.TEST_END,
        mtm.TEST_DESCRIPTION,
        CAST(FLOOR((session_repair.SESSION_START_DT - mtm.TEST_START) / 7) AS INTEGER) AS WEEK_FROM_TEST_START,
        SUM(CASE WHEN session_repair.CIBOODLE_RFC_LEVEL_1_KEY IN (2,3) THEN 1 ELSE 0 END) AS OPPORTUNITY,
        SUM(CASE WHEN session_repair.CIBOODLE_RFC_LEVEL_1_KEY = 3 THEN 1 ELSE 0 END) AS SNS
    FROM 
        FINANCE_ANALYTICS.ADHOC_TBLS.MARKET_TESTING_METADATA mtm
        INNER JOIN PRD_MSO.HISTORICAL.RFC_CIBOODLE_SESSION_REPAIR_CRT_SVC session_repair
        ON session_repair.ZIP_CD = mtm.ZIP_CODE
        AND session_repair.SESSION_START_DT BETWEEN mtm.TEST_START - 35 AND mtm.TEST_END
        WHERE session_repair.CIBOODLE_ACTION_CD IN ('A SOC CC','A NO SVC')
    GROUP BY 
        1, 2, 3, 4, 5, 6, 7, 8, 9, 10
)
SELECT 
    YYYYWW,
    mtm.TEST_NAME,
    mtm.TEST_CATEGORY,
    mtm.TREATMENT,
    mtm.ZIP_CODE,
    mtm.PRICING_MARKET,
    mtm.TEST_CELL,
    mtm.TEST_START,
    mtm.TEST_END,
    mtm.TEST_DESCRIPTION,
    COALESCE(cd.WEEK_FROM_TEST_START, sns.WEEK_FROM_TEST_START) AS WEEK_FROM_TEST_START,
    creates.CREATES,
    creates.CANCELS_7DAY,
    sns.OPPORTUNITY,
    sns.SNS
FROM FINANCE_ANALYTICS.ADHOC_TBLS.MARKET_TESTING_METADATA m
INNER JOIN FINANCE_ANALYTICS.ADHOC_TBLS.UE_SERVICE_ORDER_PROFIT u 
    ON     mtm.ZIP_CODE = u.ZIP_CODE AND u.SO_STS_DT BETWEEN mtm.TEST_START - 35 AND mtm.TEST_END
LEFT JOIN creates 
    ON  creates.TEST_NAME = mtm.TEST_NAME
    AND creates.TEST_CATEGORY = mtm.TEST_CATEGORY
    AND creates.TREATMENT = mtm.TREATMENT
    AND creates.ZIP_CODE = mtm.ZIP_CODE
    AND creates.TEST_CELL = COALESCE(mtm.TEST_CELL, 'NA')
LEFT JOIN sns 
    ON  sns.TEST_NAME = mtm.TEST_NAME
    AND sns.TEST_CATEGORY = mtm.TEST_CATEGORY
    AND sns.TREATMENT = mtm.TREATMENT
    AND sns.ZIP_CODE = mtm.ZIP_CODE
    AND sns.TEST_CELL = COALESCE(mtm.TEST_CELL, 'NA')
    AND sns.WEEK_FROM_TEST_START = CAST(FLOOR((u.SO_STS_DT - mtm.TEST_START) / 7) AS INTEGER)
GROUP BY 
    1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16
```

## Executable query

> Note: runs on Snowflake

```sql
SELECT YYYYWW
 ,MARKET_TESTING_METADATA.TEST_NAME
 ,MARKET_TESTING_METADATA.TEST_CATEGORY
 ,MARKET_TESTING_METADATA.TREATMENT
 ,REPLACE(MARKET_TESTING_METADATA.PRICING_MARKET, '_Test','') AS PRICING_MARKET_CLEAN
 ,MARKET_TESTING_METADATA.PRICING_MARKET
 ,MARKET_TESTING_METADATA.TEST_CELL
 ,MARKET_TESTING_METADATA.TEST_START
 ,MARKET_TESTING_METADATA.TEST_END
 ,MARKET_TESTING_METADATA.TEST_DESCRIPTION
 ,CASE WHEN CAST(FLOOR((UE_COMPLETES.SO_STS_DT - MARKET_TESTING_METADATA.TEST_START)/7) AS INTEGER) >= 0 THEN CAST(FLOOR((UE_COMPLETES.SO_STS_DT - MARKET_TESTING_METADATA.TEST_START)/7) AS INTEGER) + 1 ELSE CAST(FLOOR((UE_COMPLETES.SO_STS_DT - MARKET_TESTING_METADATA.TEST_START)/7) AS INTEGER) END AS WEEK_FROM_TEST_START
 ,CASE WHEN (CASE WHEN CAST(FLOOR((UE_COMPLETES.SO_STS_DT - MARKET_TESTING_METADATA.TEST_START)/7) AS INTEGER) >= 0 THEN CAST(FLOOR((UE_COMPLETES.SO_STS_DT - MARKET_TESTING_METADATA.TEST_START)/7) AS INTEGER) + 1 ELSE CAST(FLOOR((UE_COMPLETES.SO_STS_DT - MARKET_TESTING_METADATA.TEST_START)/7) AS INTEGER) END) < 0 THEN 'PreTest' ELSE 'PostTest' END AS TEST_PRE_POST
 ,UE_CREATES.CREATES AS CREATES
 ,UE_CREATES.CANCELS_7DAY AS CANCELS_7DAY
 ,MSO_SNS.OPPORTUNITY
 ,MSO_SNS.SNS
 ,COUNT(DISTINCT UE_COMPLETES.SO_NO||UE_COMPLETES.SVC_UN_NO) AS COMPLETE_VOLUME
 ,COUNT(DISTINCT CASE WHEN UE_COMPLETES.SO_STS_DESC = 'ED - Estimate Declined' THEN UE_COMPLETES.SO_NO||UE_COMPLETES.SVC_UN_NO END) AS ED_VOLUME
 ,COUNT(DISTINCT CASE WHEN UE_COMPLETES.SO_STS_DESC = 'CO - Complete' THEN UE_COMPLETES.SO_NO||UE_COMPLETES.SVC_UN_NO END) AS REAPAIR_VOLUME
 ,SUM(CASE WHEN UE_COMPLETES.SO_STS_DESC = 'ED - Estimate Declined' THEN UE_COMPLETES.CC_REV + UE_COMPLETES.CM_CC_REV+UE_COMPLETES.CLM_LAB_REV+UE_COMPLETES.CLM_PRT_REV+UE_COMPLETES.CLM_OTHER_REV END) AS ED_REVENUE
 ,SUM(CASE WHEN UE_COMPLETES.SO_STS_DESC = 'CO - Complete' THEN UE_COMPLETES.CC_REV + UE_COMPLETES.CM_CC_REV+UE_COMPLETES.CLM_LAB_REV+UE_COMPLETES.CLM_PRT_REV+UE_COMPLETES.CLM_OTHER_REV END) AS REPAIR_REVENUE
 ,SUM(UE_COMPLETES.CC_REV + UE_COMPLETES.CM_CC_REV + UE_COMPLETES.CLM_LAB_REV + UE_COMPLETES.CLM_PRT_REV + UE_COMPLETES.CLM_OTHER_REV) AS REVENUE
 ,SUM(CASE WHEN UE_COMPLETES.SO_STS_DESC = 'ED - Estimate Declined' THEN UE_COMPLETES.VARIABLE_PROFIT END) aS ED_VARIABLE_PROFIT
 ,SUM(CASE WHEN UE_COMPLETES.SO_STS_DESC = 'CO - Complete' THEN UE_COMPLETES.VARIABLE_PROFIT END) AS REPAIR_VARIABLE_PROFIT
 ,SUM(UE_COMPLETES.VARIABLE_PROFIT + MARKETING_VARIABLE + MARKETING_FIXED) aS VARIABLE_PROFIT
FROM FINANCE_ANALYTICS.ADHOC_TBLS.MARKET_TESTING_METADATA
 INNER JOIN FINANCE_ANALYTICS.ADHOC_TBLS.UE_SERVICE_ORDER_PROFIT AS UE_COMPLETES ON MARKET_TESTING_METADATA.ZIP_CODE = UE_COMPLETES.ZIP_CODE
 AND UE_COMPLETES.SO_STS_DT >= MARKET_TESTING_METADATA.TEST_START - 35
 AND UE_COMPLETES.SO_STS_DT <= MARKET_TESTING_METADATA.TEST_END
 AND CALL_TYPE = 'D2C'
 AND SO_STS_DESC IN ('CO - Complete','ED - Estimate Declined')
 AND CASE WHEN MARKET_TESTING_METADATA.PRODUCT_CAT IS NULL AND MARKET_TESTING_METADATA.DIFFICULTY_LEVEL IS NULL THEN 1
 WHEN MARKET_TESTING_METADATA.PRODUCT_CAT IS NULL AND UPPER(MARKET_TESTING_METADATA.DIFFICULTY_LEVEL) = UPPER(UE_COMPLETES.DIFFICULTY_LEVEL) THEN 1
 WHEN MARKET_TESTING_METADATA.DIFFICULTY_LEVEL IS NULL AND UPPER(MARKET_TESTING_METADATA.PRODUCT_CAT) = UPPER(UE_COMPLETES.PRODCAT) THEN 1
 WHEN UPPER(MARKET_TESTING_METADATA.DIFFICULTY_LEVEL) = UPPER(UE_COMPLETES.DIFFICULTY_LEVEL) 
AND UPPER(MARKET_TESTING_METADATA.PRODUCT_CAT) = UPPER(UE_COMPLETES.PRODCAT) THEN 1 ELSE 0 END = 1  
 --Creates Data  
 LEFT JOIN (SELECT MARKET_TESTING_METADATA.TEST_NAME
 ,MARKET_TESTING_METADATA.TEST_CATEGORY
 ,MARKET_TESTING_METADATA.TREATMENT
 ,MARKET_TESTING_METADATA.ZIP_CODE
 ,MARKET_TESTING_METADATA.PRICING_MARKET
 ,MARKET_TESTING_METADATA.TEST_CELL
 ,MARKET_TESTING_METADATA.TEST_START
 ,MARKET_TESTING_METADATA.TEST_END
 ,MARKET_TESTING_METADATA.TEST_DESCRIPTION
 ,CAST(FLOOR((UE_CREATES.SO_CRT_DT - MARKET_TESTING_METADATA.TEST_START)/7) AS INTEGER) AS WEEK_FROM_TEST_START
 ,COUNT(DISTINCT UE_CREATES.SO_NO || UE_CREATES.SVC_UN_NO) AS CREATES
 ,COUNT(DISTINCT CASE WHEN UE_CREATES.SO_STS_DESC = 'CA - Cancelled' AND UE_CREATES.CYCLE_TIME <= 7 THEN UE_CREATES.SO_NO || UE_CREATES.SVC_UN_NO END) AS CANCELS_7DAY
 FROM FINANCE_ANALYTICS.ADHOC_TBLS.MARKET_TESTING_METADATA
 INNER JOIN FINANCE_ANALYTICS.ADHOC_TBLS.UE_SERVICE_ORDER_PROFIT AS UE_CREATES ON MARKET_TESTING_METADATA.ZIP_CODE = UE_CREATES.ZIP_CODE
 AND UE_CREATES.SO_CRT_DT >= MARKET_TESTING_METADATA.TEST_START - 35
 AND UE_CREATES.SO_CRT_DT <= MARKET_TESTING_METADATA.TEST_END
 AND CALL_TYPE = 'D2C'
 AND CASE WHEN MARKET_TESTING_METADATA.PRODUCT_CAT IS NULL THEN 1
 WHEN UPPER(MARKET_TESTING_METADATA.PRODUCT_CAT) = UPPER(UE_CREATES.PRODCAT) THEN 1 ELSE 0 END = 1  
 GROUP BY 1,2,3,4,5,6,7,8,9,10
 ) AS UE_CREATES ON UE_CREATES.TEST_NAME = MARKET_TESTING_METADATA.TEST_NAME
 AND UE_CREATES.TEST_CATEGORY = MARKET_TESTING_METADATA.TEST_CATEGORY
 AND UE_CREATES.TREATMENT = MARKET_TESTING_METADATA.TREATMENT
 AND UE_CREATES.ZIP_CODE = MARKET_TESTING_METADATA.ZIP_CODE
 AND CASE WHEN UE_CREATES.TEST_CELL IS NULL THEN 'NA' ELSE UE_CREATES.TEST_CELL END = CASE WHEN MARKET_TESTING_METADATA.TEST_CELL IS NULL THEN 'NA' ELSE MARKET_TESTING_METADATA.TEST_CELL END
 AND UE_CREATES.WEEK_FROM_TEST_START = CAST(FLOOR((UE_COMPLETES.SO_STS_DT - MARKET_TESTING_METADATA.TEST_START)/7) AS INTEGER)
 --MSO Service Not Set Data
 LEFT JOIN (SELECT MARKET_TESTING_METADATA.TEST_NAME
 ,MARKET_TESTING_METADATA.TEST_CATEGORY
 ,MARKET_TESTING_METADATA.TREATMENT
 ,MARKET_TESTING_METADATA.ZIP_CODE
 ,MARKET_TESTING_METADATA.PRICING_MARKET
 ,MARKET_TESTING_METADATA.TEST_CELL
 ,MARKET_TESTING_METADATA.TEST_START
 ,MARKET_TESTING_METADATA.TEST_END
 ,MARKET_TESTING_METADATA.TEST_DESCRIPTION
 ,CAST(FLOOR((RFC_CIBOODLE_SESSION_REPAIR_CRT_SVC.SESSION_START_DT - MARKET_TESTING_METADATA.TEST_START)/7) AS INTEGER) AS WEEK_FROM_TEST_START
 ,SUM(CASE WHEN CIBOODLE_RFC_LEVEL_1_KEY IN (2,3) THEN 1 ELSE 0 END) AS OPPORTUNITY
 ,SUM(CASE WHEN CIBOODLE_RFC_LEVEL_1_KEY = 3 THEN 1 ELSE 0 END) AS SNS
 FROM FINANCE_ANALYTICS.ADHOC_TBLS.MARKET_TESTING_METADATA
 INNER JOIN PRD_MSO.HISTORICAL.RFC_CIBOODLE_SESSION_REPAIR_CRT_SVC ON RFC_CIBOODLE_SESSION_REPAIR_CRT_SVC.ZIP_CD = MARKET_TESTING_METADATA.ZIP_CODE
 AND RFC_CIBOODLE_SESSION_REPAIR_CRT_SVC.SESSION_START_DT >= MARKET_TESTING_METADATA.TEST_START - 35
 AND RFC_CIBOODLE_SESSION_REPAIR_CRT_SVC.SESSION_START_DT <= MARKET_TESTING_METADATA.TEST_END
 WHERE CIBOODLE_ACTION_CD IN ('A SOC CC','A NO SVC')
 GROUP BY 1,2,3,4,5,6,7,8,9,10
 ) AS MSO_SNS ON MSO_SNS.TEST_NAME = MARKET_TESTING_METADATA.TEST_NAME
 AND MSO_SNS.TEST_CATEGORY = MARKET_TESTING_METADATA.TEST_CATEGORY
 AND MSO_SNS.TREATMENT = MARKET_TESTING_METADATA.TREATMENT
 AND MSO_SNS.ZIP_CODE = MARKET_TESTING_METADATA.ZIP_CODE
 AND CASE WHEN UE_CREATES.TEST_CELL IS NULL THEN 'NA' ELSE UE_CREATES.TEST_CELL END = CASE WHEN MARKET_TESTING_METADATA.TEST_CELL IS NULL THEN 'NA' ELSE MARKET_TESTING_METADATA.TEST_CELL END
 AND MSO_SNS.WEEK_FROM_TEST_START = CAST(FLOOR((UE_COMPLETES.SO_STS_DT - MARKET_TESTING_METADATA.TEST_START)/7) AS INTEGER)

GROUP BY 1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16
```

---

*Document last updated*: 2023-Sep-29
