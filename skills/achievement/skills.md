---
name: achievement
description: 當用戶提到「Target vs Achieve」、「Achievement Rate]使用此技能
-----
**Sql template**
WITH actual AS (
    SELECT 
        data_type as period,
        SUM(value) as actual_sales
    FROM netsuite.optw_dw_dsi_monthly_data_v
    WHERE section = 'TOTAL SALES'
        AND data_type >= '202401'
    GROUP BY data_type
),
forecast AS (
    SELECT 
        SUBSTRING(data_type, 1, 6) as period,
        SUM(value) as forecast_sales
    FROM netsuite.optw_dw_dsi_monthly_data_v
    WHERE section = 'Sales Forecast'
    GROUP BY SUBSTRING(data_type, 1, 6)
)
SELECT 
    COALESCE(a.period, f.period) as period,
    a.actual_sales,
    f.forecast_sales,
    (a.actual_sales - f.forecast_sales) as variance,
    CASE 
        WHEN a.actual_sales > 0 THEN 
            ROUND((a.actual_sales - f.forecast_sales) * 100.0 / a.actual_sales, 2)
        ELSE NULL 
    END as variance_pct
FROM actual a
FULL OUTER JOIN forecast f ON a.period = f.period
ORDER BY period
