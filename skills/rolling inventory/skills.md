---
name: psi_rolling_forecast
description: 執行進階的 PSI (進銷存) 滾動庫存預測,當用戶提到「供應與需求是否平衡」、「Sales forecast or purchase forecast 異常」或「庫存是不是過高或過低」,[是否有呆滯庫存要處理]時，使用此技能
-----
# 技能說明
依據核心商務邏輯產生正確的SQL

## 核心商務邏輯
- **計算公式**：期末庫存 = 期初庫存 + 供應 - 需求(t)。
- **常用模板**：table or view : optw_dw_dsi_monthly_data_v
- **查詢型號以secondary_model為主** : Where secondary_model like 'model_name'


## Steps
**Step 1: base_inventory CTE**
- Get Begin Inventory where `data_source = 'Begin Inventory'` AND `data_type = 'FG + In Transit'`
- Use MAX(bucket_date) from last complete month
- GROUP BY region, secondary_model, bucket_date

**Step 2: forecast_months CTE**
- Generate 8 month buckets using generate_series(0, 7)

**Step 3: sales_data CTE**
- Aggregate Sales Forecast by bucket_date
- Filter: `data_source = 'Sales Forecast'`, bucket_date range: current + 7 months
- GROUP BY region, secondary_model, bucket_date

**Step 4: purchase_data CTE**
- Aggregate ETA Purchase Forecast by bucket_date
- Filter: `data_source = 'Purchase Forecast(ETA)'`, same date range
- GROUP BY region, secondary_model, bucket_date

**Step 5: combined_data CTE**
- LEFT JOIN all CTEs on region, secondary_model, bucket_date
- COALESCE all NULL values to 0

**Step 6: running_inventory CTE**
- Use LAG() to get previous end_inv as current begin_inv
- Calculate: begin_inv + purchase - sales = end_inv
- Format bucket as YYYYMM

**Step 7: Final SELECT**
- Select with display labels, ORDER BY region, bucket

## Output
-- bucket_date	, display label : "Bucket"
-- begin_inventory	, display label : "Begin Inv."
-- eta_purchase_forecast	, display label : "Purchase FCST(ETA)"
-- sales_forecast	, display label : "Sales FCST"
-- end_inventory, display label : "End Inv."
