---
name: sales trend
description: 當用戶提到「sales trend」、「銷售趨勢]使用此技能
-----
**key logic**
-- Use data source are sold history and sales forecast with date bucket as sales quantity of secondary model {XXXX}
-- If sold history and sales forecast in same date bucket then use sales forecast as sales quantity 
-- if user not mention the period, just query all available period data
-- summary the sales trend
**SQL template**
WITH ranked_data AS (
    SELECT 
        data_type, 
        SUM(value) as total_sales, 
        data_source,
        ROW_NUMBER() OVER (
            PARTITION BY LEFT(data_type, 6)  -- Extract just YYYYMM
            ORDER BY CASE WHEN data_source = 'Sold History' THEN 1 ELSE 2 END
        ) as rn
    FROM netsuite.optw_dw_dsi_monthly_data_v 
    WHERE data_source IN ('Sold History', 'Sales Forecast') 
        AND data_type BETWEEN 'xxxxxxx' AND 'xxxxxx' 
    GROUP BY data_type, data_source
)
SELECT LEFT(data_type, 6) as data_type, total_sales, data_source
FROM ranked_data
WHERE rn = 1
ORDER BY data_type
