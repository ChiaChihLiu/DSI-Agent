---
name: sales trend
description: 當用戶提到「sales trend」、「銷售趨勢]使用此技能
-----
**key logic**
-- Use data source are sold history and sales forecast with date bucket as sales quantity of secondary model {XXXX}
-- If sold history and sales forecast in same date bucket then use sales forecast as sales quantity 
-- if user not mention the period, just query all available period data
**SQL template**
WITH history AS (
    SELECT data_type, SUM(value) as total_sales, data_source
    FROM netsuite.optw_dw_dsi_monthly_data_v
    WHERE data_source = 'Sold History'
    GROUP BY data_type, data_source
),
forecast AS (
    SELECT data_type, SUM(value) as total_sales, data_source
    FROM netsuite.optw_dw_dsi_monthly_data_v
    WHERE data_source = 'Sales Forecast'
    GROUP BY data_type, data_source
)
SELECT 
    COALESCE(h.data_type, f.data_type) as data_type,
    COALESCE(h.total_sales, f.total_sales) as total_sales,
    COALESCE(h.data_source, f.data_source) as data_source
FROM history h
FULL OUTER JOIN forecast f ON h.data_type = f.data_type
ORDER BY data_type;
