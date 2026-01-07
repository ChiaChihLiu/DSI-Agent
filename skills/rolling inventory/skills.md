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
- **常用模板**：table or view : optw_dw_dsi_monthly_data_v
- **查詢型號以secondary_model為主** : Where secondary_model like 'model_name'
## Steps
- Collect data by secondary model is {XXXXXX} for each region,
- Only data source is "Begin Inventory" with data type ONLY "FG + In Transit" as "Begin Inventory", its bucket date is "Start Date" and include data source are "Sales Forecast" , "ETA Purchase Forecast" with bucket date after "Start Date" 
- use "Begin Inventory" with "Sales Forecast" and "ETA Purchase Forecast" with arrived bucket date to calculate "End Inventory", "End Inventory" of this bucket date will be the next bucket date "Begin Inventory" show me output for detail table with border of running inventory, 
- show result table with border

```
