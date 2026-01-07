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
- 
## SQL Query Template

### 查詢結構 (Query Structure)

**Step 1 - 參考日期**
- 目的：找出有效的基準日期
- 技術：WHERE 篩選 + ORDER BY + LIMIT 1
- 輸出：cutoff_date, cutoff_date_str

**Step 2 - 期初狀態**
- 目的：計算起始庫存/餘額
- 技術：JOIN Step1 + SUM + GROUP BY
- 輸出：initial_inventory, inventory_date

**Step 3 - 期間數據**
- 目的：收集各期流入(供應)與流出(需求)
- 技術：UNION ALL 合併多來源 + 時間位移(如適用) + GROUP BY 彙總
- 輸出：period, demand, supply

**Step 4 - 累積計算**
- 目的：計算淨變動與滾動累計
- 技術：Window Function - SUM() OVER (ORDER BY period ROWS UNBOUNDED PRECEDING)
- 輸出：net_change, cumulative_net

**Step 5 - 最終輸出**
- 目的：組合所有數據 + 狀態判斷 + 建議
- 技術：CROSS JOIN Step2 + LAG() 取前期 + CASE 門檻判斷
- 輸出：期初庫存、需求、供應、期末庫存、狀態、建議量

## 關鍵公式
- 期初庫存 = initial_inventory + LAG(cumulative_net)
- 期末庫存 = initial_inventory + cumulative_net
- 淨變動 = supply - demand

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
