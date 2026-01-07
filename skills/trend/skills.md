---
name: sales trend
description: 當用戶提到「sales trend」、「銷售趨勢]使用此技能
-----
**key logic**
-- Use data source are sold history and sales forecast with date bucket as sales quantity of secondary model {XXXX}
-- If sold history and sales forecast in same date bucket then use sales forecast as sales quantity 
-- if user not mention the period, just query all available period data
**SQL template**
SELECT 
    data_type,
    SUM(value) as total_sales
FROM netsuite.optw_dw_dsi_monthly_data_v
WHERE data_source in ('Sold History','Sales Forecast')
    AND data_type BETWEEN 'YYYYMM' AND 'YYYYMM'
GROUP BY data_type
ORDER BY data_type;
