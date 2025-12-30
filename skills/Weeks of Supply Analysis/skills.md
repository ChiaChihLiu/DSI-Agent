---
name: Weeks of Supply Analysis
description: 當用戶提到 "WOS" / "供應週數" / "呆滯" / "overstockFWOS" / "weeks of supply"時使用此技能
-----
# 技能說明
依據核心商務邏輯產生正確的SQL

## 核心商務邏輯
- **FWOS計算** = 當前庫存 / (未來3個月平均預測 / 4.33週)
- **FWOS < 4週** → 🔴 缺貨警報（緊急採購建議）
- **FWOS 4-8週** → 🟡 正常範圍
- **FWOS 8-12週** → 🟢 健康水平
- **FWOS > 12週** → 🟠 呆滯警報（**系統新增功能**）
- 呆滯庫存建議促銷折扣比例基於超額週數計算
- 缺貨建議採購數量基於短缺週數 × 週度平均需求
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
    -- 計算當前庫存：只用 FG + In Transit
    SELECT
        SUM(CASE WHEN t.data_type = 'FG + In Transit' THEN t.value ELSE 0 END) as total_inventory,
        l.cutoff_date_str as inventory_date
    FROM latest_valid_inventory_date l
    JOIN netsuite.optw_dw_dsi_st t
        ON t.section = l.section
        AND t.data_type = 'FG + In Transit'
    GROUP BY l.cutoff_date_str
),
future_forecast AS (
    -- 計算未來3個月平均預測
    SELECT
        AVG(monthly_forecast) as avg_monthly_forecast
    FROM (
        SELECT
            SUBSTRING(data_type, 1, 6) as period,
            SUM(value) as monthly_forecast
        FROM netsuite.optw_dw_dsi_st
        WHERE section = 'Sales Forecast'
            AND data_type BETWEEN TO_CHAR(CURRENT_DATE, 'YYYYMM')
            AND TO_CHAR(ADD_MONTHS(CURRENT_DATE, 3), 'YYYYMM')
        GROUP BY SUBSTRING(data_type, 1, 6)
    ) subq
),
wos_calculation AS (
    SELECT
        i.inventory_date,
        i.total_inventory,
        f.avg_monthly_forecast,
        -- 計算週度平均（1個月 ≈ 4.33週）
        ROUND(f.avg_monthly_forecast / 4.33, 2) as avg_weekly_forecast,
        -- FWOS = 當前庫存 / 週度平均預測
        ROUND(
            i.total_inventory / NULLIF(f.avg_monthly_forecast / 4.33, 0),
            1
        ) as fwos_weeks
    FROM current_inventory i
    CROSS JOIN future_forecast f
)
SELECT
    inventory_date as 庫存基準日,
    ROUND(total_inventory, 0) as 當前庫存,
    ROUND(avg_monthly_forecast, 0) as 未來3個月平均月需求,
    avg_weekly_forecast as 平均週需求,
    fwos_weeks as 前瞻供應週數_FWOS,
    -- 第二步核心邏輯：WOS警報分級
    CASE
        WHEN fwos_weeks < 4 THEN '🔴 缺貨警報'
        WHEN fwos_weeks >= 4 AND fwos_weeks < 8 THEN '🟡 正常'
        WHEN fwos_weeks >= 8 AND fwos_weeks < 12 THEN '🟢 健康'
        WHEN fwos_weeks >= 12 THEN '🟠 呆滯警報 (Slow-Moving)'
        ELSE '⚪ 資料不足'
    END as WOS狀態,
    -- 警報類型
    CASE
        WHEN fwos_weeks < 4 THEN '缺貨風險'
        WHEN fwos_weeks >= 12 THEN '呆滯庫存'
        ELSE NULL
    END as 警報類型,
    -- 建議行動
    CASE
        WHEN fwos_weeks >= 12 THEN
            '💡 建議促銷或調撥，暫停採購 | Discount: ' ||
            ROUND((fwos_weeks - 8) * 100.0 / fwos_weeks, 0) || '%'
        WHEN fwos_weeks < 4 THEN
            '🚨 緊急採購建議: ' ||
            ROUND((4 - fwos_weeks) * avg_weekly_forecast, 0) || ' units'
        ELSE '✓ 維持現狀'
    END as 建議行動,
    -- 優先級
    CASE
        WHEN fwos_weeks < 2 THEN '1 - 緊急缺貨'
        WHEN fwos_weeks < 4 THEN '2 - 缺貨風險'
        WHEN fwos_weeks >= 12 AND fwos_weeks < 16 THEN '2 - 呆滯風險'
        WHEN fwos_weeks >= 16 THEN '1 - 嚴重呆滯'
        ELSE '3 - 正常'
    END as 優先級
FROM wos_calculation;
```



