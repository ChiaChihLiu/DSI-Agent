---
name: Variance Analysis
description: 自動偵測需求激增/積壓 (±20% thresholds) 追逐策略 vs 平準策略自動推薦 OTB額度管理與重新分配建議

# 滾動預測 SOP
用戶問變異分析" / "需求激增" / "追逐策略" / "庫存積壓" / "OTB"，使用此技能。

**說明：**
- **第一步：變異分析** - 比較實際銷售 vs 預測銷售
- **Variance > +20%** → 需求激增，建議啟用追逐策略（Chase Strategy），增加採購量
- **Variance < -20%** → 庫存積壓風險，建議減少採購或促銷，計算OTB額度凍結影響
- **Variance -10% to +10%** → 正常範圍，維持平準策略（Level Strategy）
- 採購調整建議為變異數的50%（保守估計）
- OTB影響：負向變異凍結額度，正向變異需要額外預算
- 用於驅動後續的WOS分析（第二步）和動態採購（第三步）

## SQL template 
## SQL should match the Redshift

WITH actual_sales AS (
    SELECT 
        data_type as period, 
        SUM(value) as actual_sales 
    FROM netsuite.optw_dw_dsi_st 
    WHERE section = 'TOTAL SALES' 
    AND data_type >= TO_CHAR(ADD_MONTHS(CURRENT_DATE, -6), 'YYYYMM') -- Last 6 months
    AND secondary_model LIKE '%ML1080ST%' 
    AND region = 'EMEA' 
    GROUP BY data_type
), 
sales_forecast AS (
    SELECT 
        SUBSTRING(data_type, 1, 6) as period, 
        SUM(value) as forecast_sales 
    FROM netsuite.optw_dw_dsi_st 
    WHERE section = 'Sales Forecast' 
    AND SUBSTRING(data_type, 1, 6) >= TO_CHAR(ADD_MONTHS(CURRENT_DATE, -6), 'YYYYMM') 
    AND secondary_model LIKE '%ML1080ST%' 
    AND region = 'EMEA' 
    GROUP BY SUBSTRING(data_type, 1, 6)
), 
variance_analysis AS (
    SELECT 
        COALESCE(a.period, f.period) as period, 
        a.actual_sales, 
        f.forecast_sales, 
        (a.actual_sales - f.forecast_sales) as variance, 
        CASE 
            WHEN f.forecast_sales > 0 THEN ROUND((a.actual_sales - f.forecast_sales) * 100.0 / f.forecast_sales, 2) 
            ELSE NULL 
        END as variance_pct 
    FROM actual_sales a 
    FULL OUTER JOIN sales_forecast f ON a.period = f.period
)
SELECT 
    period, 
    ROUND(actual_sales, 0) as actual_sales, 
    ROUND(forecast_sales, 0) as forecast_sales, 
    ROUND(variance, 0) as variance_qty, 
    variance_pct as variance_percentage,
    CASE 
        WHEN variance_pct > 20 THEN 'Aggressive Inventory Increase (Chase Strategy)' 
        WHEN variance_pct >= 10 AND variance_pct <= 20 THEN 'Moderate Inventory Increase'
        WHEN variance_pct <= -20 THEN 'Aggressive Inventory Reduction (Risk Mitigation)'
        WHEN variance_pct <= -10 AND variance_pct > -20 THEN 'Moderate Inventory Reduction'
        ELSE 'Maintain Current Level' 
    END as procurement_advice,
    CASE 
        WHEN variance_pct > 20 THEN CONCAT('+', ROUND(variance_pct / 2, 0), '% Increase Procurement') 
        WHEN variance_pct < -20 THEN CONCAT(ROUND(variance_pct / 2, 0), '% Decrease Procurement') 
        ELSE 'Maintain Status Quo' 
    END as procurement_adjustment,
    CASE 
        WHEN variance_pct > 20 THEN CONCAT('OTB Budget Increase: ', ROUND(variance / 2, 0), ' units') 
        ELSE NULL 
    END as otb_impact,
    CASE 
        WHEN ABS(variance_pct) > 30 THEN '1 - Urgent' 
        WHEN ABS(variance_pct) > 20 THEN '2 - High' 
        WHEN ABS(variance_pct) > 10 THEN '3 - Medium' 
        ELSE '4 - Low' 
    END as priority_level
FROM variance_analysis 
WHERE actual_sales IS NOT NULL 
ORDER BY period DESC
```


---
