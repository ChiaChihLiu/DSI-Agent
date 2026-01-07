---
name: psi_rolling_forecast
description: 執行進階的 PSI (進銷存) 滾動庫存預測,當用戶提到「供應與需求是否平衡」、「Sales forecast or purchase forecast 異常」或「庫存是不是過高或過低」,[是否有呆滯庫存要處理]時，使用此技能
-----
# 技能說明
依據核心商務邏輯產生正確的SQL

## 核心商務邏輯
- **計算公式**：期末庫存 = 期初庫存 + 供應 - 需求(t)。
- **輸出規範**：必須遵循標準 9 欄位格式（期間/基準日/期初/需求/供應/月淨變動/期末/狀態/建議採購）。
- **常用模板**：SQL template。
- 
# 滾動預測 SOP
- **Step1**： 了解用戶問題和意圖
- **Step2**： 了解 database or tables or import logic,if need you can Call the 'Call 'My Sub-Workflow 2' to know the basic infromation
- **Step3**： 了解滾動庫存預測核心邏輯
              -- 1. 上期期末庫存 = 本期期初庫存
              -- 2. 期初庫存只用 FG + In Transit
              -- 3. 庫存基準日取上一個月為當月的庫存基準日
## SQL template
WITH latest_valid_inventory_date AS (
    SELECT
        section,
        SUBSTRING(section, 24) as cutoff_date_str,
        TO_DATE(SUBSTRING(section, 24), 'DD-MON-YY') as cutoff_date
    FROM netsuite.optw_dw_dsi_st
    WHERE section LIKE 'Inventory cut off date:%'
        AND TO_DATE(SUBSTRING(section, 24), 'DD-MON-YY')
            BETWEEN DATE_TRUNC('month', ADD_MONTHS(CURRENT_DATE, -1)::TIMESTAMP)::DATE
            AND LAST_DAY(ADD_MONTHS(CURRENT_DATE, -1))
    ORDER BY cutoff_date DESC
    LIMIT 1
),
current_inventory AS (
    SELECT
        SUM(CASE WHEN t.data_type = 'FG + In Transit' THEN t.value ELSE 0 END) as initial_inventory,
        l.cutoff_date_str as inventory_date
    FROM latest_valid_inventory_date l
    JOIN netsuite.optw_dw_dsi_st t
        ON t.section = l.section
        AND t.data_type = 'FG + In Transit'
    GROUP BY l.cutoff_date_str
),
monthly_forecast AS (
    -- Sales Forecast: 當月需求
    SELECT
        SUBSTRING(data_type, 1, 6) as period,
        SUM(value) as demand,
        0 as supply
    FROM netsuite.optw_dw_dsi_st
    WHERE section = 'Sales Forecast'
        AND data_type >= TO_CHAR(CURRENT_DATE, 'YYYYMM')
    GROUP BY SUBSTRING(data_type, 1, 6)

    UNION ALL

    -- Purchase Forecast: 當月供應 (使用 ETA)
    SELECT
        SUBSTRING(data_type, 1, 6) as period,
        0 as demand,
        SUM(value) as supply
    FROM netsuite.optw_dw_dsi_st
    WHERE section = 'Purchase Forecast(ETA)'
        AND data_type >= TO_CHAR(CURRENT_DATE, 'YYYYMM')
    GROUP BY SUBSTRING(data_type, 1, 6)
),
monthly_forecast_aggregated AS (
    SELECT
        period,
        SUM(demand) as demand,
        SUM(supply) as supply
    FROM monthly_forecast
    GROUP BY period
),
forecast_with_cumulative AS (
    SELECT
        period,
        demand,
        supply,
        (supply - demand) as net_change,
        SUM(supply - demand) OVER (
            ORDER BY period
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) as cumulative_net
    FROM monthly_forecast_aggregated
)
SELECT
    f.period as 期間,
    i.inventory_date as 庫存基準日,
    ROUND(
        i.initial_inventory + COALESCE(LAG(f.cumulative_net) OVER (ORDER BY f.period), 0),
        0
    ) as 期初庫存,
    ROUND(f.demand, 0) as 需求,
    ROUND(f.supply, 0) as 供應,
    ROUND(f.net_change, 0) as 月淨變動,
    ROUND(
        i.initial_inventory + f.cumulative_net,
        0
    ) as 預計期末庫存,
    CASE
        WHEN i.initial_inventory + f.cumulative_net < 0 THEN '🔴 預計缺貨'
        WHEN i.initial_inventory + f.cumulative_net < 30 THEN '🟡 低庫存警告'
        WHEN i.initial_inventory + f.cumulative_net < 60 THEN '🟢 正常'
        ELSE '🟢 健康'
    END as 庫存狀態,
    CASE
        WHEN i.initial_inventory + f.cumulative_net < 30 AND f.demand > 0
        THEN ROUND(60 - (i.initial_inventory + f.cumulative_net), 0)
        ELSE NULL
    END as 建議採購量
FROM forecast_with_cumulative f
CROSS JOIN current_inventory i
ORDER BY f.period;
```


