---
name: Variance Analysis
description: 用戶問[變異分析],[OTB]時，使用此技能
---

## 滾動預測 SOP


## SQL template 
WITH actual_sales AS (
    SELECT
        data_type as period,
        SUM(value) as actual_sales
    FROM netsuite.optw_dw_dsi_st
    WHERE section = 'TOTAL SALES'
        AND data_type >= TO_CHAR(ADD_MONTHS(CURRENT_DATE, -6), 'YYYYMM')  -- Last 6 months
    GROUP BY data_type
),
sales_forecast AS (
    SELECT
        SUBSTRING(data_type, 1, 6) as period,
        SUM(value) as forecast_sales
    FROM netsuite.optw_dw_dsi_st
    WHERE section = 'Sales Forecast'
        AND SUBSTRING(data_type, 1, 6) >= TO_CHAR(ADD_MONTHS(CURRENT_DATE, -6), 'YYYYMM')
    GROUP BY SUBSTRING(data_type, 1, 6)
),
variance_analysis AS (
    SELECT
        COALESCE(a.period, f.period) as period,
        a.actual_sales,
        f.forecast_sales,
        (a.actual_sales - f.forecast_sales) as variance,
        CASE
            WHEN f.forecast_sales > 0 THEN
                ROUND((a.actual_sales - f.forecast_sales) * 100.0 / f.forecast_sales, 2)
            ELSE NULL
        END as variance_pct
    FROM actual_sales a
    FULL OUTER JOIN sales_forecast f ON a.period = f.period
)
SELECT
    period as 期間,
    ROUND(actual_sales, 0) as 實際銷售,
    ROUND(forecast_sales, 0) as 預測銷售,
    ROUND(variance, 0) as 變異數量,
    variance_pct as 變異百分比,
    -- 策略建議（第一步核心邏輯）
    CASE
        WHEN variance_pct > 20 THEN '🔴 需求激增 - 啟用追逐策略 (Chase Strategy)'
        WHEN variance_pct >= 10 AND variance_pct <= 20 THEN '🟡 需求上升 - 關注趨勢'
        WHEN variance_pct >= -10 AND variance_pct < 10 THEN '🟢 正常範圍 - 維持平準策略 (Level Strategy)'
        WHEN variance_pct >= -20 AND variance_pct < -10 THEN '🟡 需求下降 - 關注趨勢'
        WHEN variance_pct < -20 THEN '🟠 庫存積壓風險 - 減少採購/啟動促銷'
        ELSE '⚪ 資料不足'
    END as 策略建議,
    -- 採購調整建議（%）- FIXED: Using || operator with ::TEXT casting
    CASE
        WHEN variance_pct > 20 THEN '+' || ROUND(variance_pct / 2, 0)::TEXT || '% 增加採購'
        WHEN variance_pct < -20 THEN ROUND(variance_pct / 2, 0)::TEXT || '% 減少採購'
        ELSE '維持現況'
    END as 採購調整建議,
    -- OTB影響（若有超額/不足）- FIXED: Using || operator with ::TEXT casting
    CASE
        WHEN variance_pct < -20 THEN
            '⚠️ 凍結OTB額度: ' || ROUND(ABS(variance), 0)::TEXT || ' units'
        WHEN variance_pct > 20 THEN
            '💡 額外OTB需求: ' || ROUND(variance / 2, 0)::TEXT || ' units'
        ELSE NULL
    END as OTB影響,
    -- 優先級
    CASE
        WHEN ABS(variance_pct) > 30 THEN '1 - 緊急'
        WHEN ABS(variance_pct) > 20 THEN '2 - 高'
        WHEN ABS(variance_pct) > 10 THEN '3 - 中'
        ELSE '4 - 低'
    END as 優先級
FROM variance_analysis
WHERE actual_sales IS NOT NULL  -- Only show periods with actual data
ORDER BY period DESC;
```
