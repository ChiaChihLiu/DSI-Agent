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
- **查詢型號以secondary_model為主**
Where secondary_model like 'model_name'
## SQL Query Template
###### Add a table alias and qualify the column reference

task_id: T1_inventory_cutoff_date
description: 從上個月的庫存截止日期中，取得最新的一筆
table: netsuite.optw_dw_dsi_monthly_data_v
filters:
  - section LIKE 'Inventory cut off date:%'
  - 日期範圍：上個月第一天 ~ 上個月最後一天
logic:
  - 從 section 欄位提取日期字串（位置24開始）
  - 轉換為日期格式 'DD-MON-YY'
  - 取最新日期（ORDER BY DESC LIMIT 1）
output_columns:
  - section (原始值)
  - cutoff_date_str (字串格式)
  - cutoff_date (日期格式)

task_id: T2_initial_inventory
description: 計算期初庫存，僅使用 FG + In Transit
depends_on: T1
table: netsuite.optw_dw_dsi_monthly_data_v
filters:
  - section = T1.section
  - data_type = 'FG + In Transit'
logic:
  - SUM(value) 作為 initial_inventory
key_rules:
  - ⚠️ 只用 FG + In Transit，不包含其他類型
output_columns:
  - initial_inventory
  - inventory_date (來自 T1)

task_id: T3_sales_forecast
description: 取得當月及未來的銷售預測作為需求
table: netsuite.optw_dw_dsi_monthly_data_v
filters:
  - section = 'Sales Forecast'
  - data_type >= 當月 (YYYYMM 格式)
logic:
  - 期間提取：SUBSTRING(data_type, 1, 6)
  - 按期間 GROUP BY 加總 value
output_columns:
  - period (YYYYMM)
  - demand (SUM of value)
  - supply = 0 (固定值)

task_id: T4_purchase_forecast
description: 取得採購預測作為供應，期間需 +1 month
table: netsuite.optw_dw_dsi_monthly_data_v
filters:
  - section = 'Purchase Forecast(ETA)'
  - data_type >= 當月 (YYYYMM 格式)
logic:
  - 按調期間 GROUP BY 加總 value

output_columns:
  - period (YYYYMM)
  - demand = 0 (固定值)
  - supply (SUM of value)

task_id: T5_merge_demand_supply
description: 將銷售預測與採購預測合併彙總
depends_on: T3, T4
logic:
  - UNION ALL T3 和 T4 的結果
  - 按 period GROUP BY
  - SUM(demand) 和 SUM(supply)
output_columns:
  - period
  - demand (彙總)
  - supply (彙總)

task_id: T6_cumulative_calculation
description: 計算月淨變動與累積淨變動
depends_on: T5
logic:
  - net_change = supply - demand
  - cumulative_net = 窗函數累加
    - SUM(supply - demand) OVER (ORDER BY period ROWS UNBOUNDED PRECEDING)
output_columns:
  - period
  - demand
  - supply
  - net_change
  - cumulative_net

task_id: T7_final_report
description: 組合所有資料，計算庫存狀態與建議採購量
depends_on: T2, T6
logic:
  期初庫存計算:
    - initial_inventory + LAG(cumulative_net, 默認0)
    - ⭐ 上期期末 = 本期期初
  
  期末庫存計算:
    - initial_inventory + cumulative_net
  
  庫存狀態判斷:
    - < 0: 🔴 預計缺貨
    - < 30: 🟡 低庫存警告
    - < 60: 🟢 正常
    - >= 60: 🟢 健康
  
  建議採購量:
    - 條件1: 期末庫存 < 30
    - 條件2: demand > 0 (⚠️ 確保有需求才建議)
    - 計算: 60 - 期末庫存

output_columns:
  - 期間
  - 庫存基準日
  - 期初庫存
  - 需求
  - 供應
  - 月淨變動
  - 預計期末庫存
  - 庫存狀態
  - 建議採購量
 

### 主查詢邏輯 (Main Query Logic)

#### 期初庫存計算
```
期初庫存 = 初始庫存 + LAG(累積淨變動)
```
- 使用 LAG() 取得上期累積淨變動
- 第一期的期初庫存 = 初始庫存

#### 期末庫存計算
```
預計期末庫存 = 初始庫存 + 本期累積淨變動
```

#### 庫存狀態判斷
- 🔴 預計缺貨: < 0
- 🟡 低庫存警告: < 30
- 🟢 正常: < 60
- 🟢 健康: >= 60

#### 建議採購量 (v1.6 邏輯)
```
當 期末庫存 < 30 且 未來有需求 (demand > 0):
  建議採購量 = 60 - 期末庫存
否則:
  建議採購量 = NULL
```


### 輸出欄位 (Output Columns)

| 欄位 | 說明 | 計算邏輯 |
|------|------|----------|
| `期間` | 預測期間 (YYYYMM) | - |
| `庫存基準日` | 庫存快照日期 | 來自 latest_valid_inventory_date |
| `期初庫存` | 期初庫存數量 | 初始庫存 + LAG(累積淨變動) |
| `需求` | Sales Forecast | 當月銷售預測 |
| `供應` | Purchase Forecast | 當月採購預測 (ETA) |
| `月淨變動` | 淨變動 | 供應 - 需求 |
| `預計期末庫存` | 預計期末庫存 | 初始庫存 + 累積淨變動 |
| `庫存狀態` | 健康度指標 | 根據期末庫存判斷 |
| `建議採購量` | 建議採購數量 | 60 - 期末庫存 (當 < 30 且有需求) |
```
