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
## SQL Query Template

### 查詢結構 (Query Structure)

#### CTE 1: latest_valid_inventory_date
- **目的**: 取得最新有效的庫存截止日期
- **邏輯**:
  - 查找上個月內的 "Inventory cut off date:%" section
  - 提取並轉換日期格式 (DD-MON-YY)
  - 取最新的一筆記錄

#### CTE 2: current_inventory
- **目的**: 計算期初庫存
- **邏輯**:
  - 僅使用 `FG + In Transit` 資料類型
  - 加總該日期的庫存數量

#### CTE 3: monthly_forecast
- **目的**: 彙總月度需求與供應
- **邏輯**:
  - **Sales Forecast**: 當月需求 (期間不變)
  - **Purchase Forecast(ETA)**: 當月供應 (期間不變)
    - 使用 ETA (預計到貨日期) 作為供應期間

#### CTE 4: monthly_forecast_aggregated
- **目的**: 按期間彙總需求與供應
- **邏輯**: GROUP BY period 加總 demand 和 supply

#### CTE 5: forecast_with_cumulative
- **目的**: 計算累積淨變動
- **邏輯**:
  - `net_change` = supply - demand (月淨變動)
  - `cumulative_net` = 累積所有期間的淨變動

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


