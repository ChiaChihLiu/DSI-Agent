## 資料來源

### Database Tables

#### table_name : optw_dw_dsi_monthly_data
- **用途**: Transaction data (交易數據)
- **說明**: 存儲月度銷售交易資料
- **關聯鍵**: `primary_model`, `secondary_model`

**欄位 (Columns):**
- `region` - 區域
- `primary_model` - 主要型號
- `secondary_model` - 次要型號
- `dmd_chip` - DMD 晶片
- `section` - 區段/類別
- `data_type` - 資料類型
- `value` - 數值
- `bucket_date` - 時間區間/日期
- `data_source` - 資料來源

####  table_name : optw_dw_dsi_model
- **用途**: Model specification (型號規格)
- **說明**: 存儲產品型號的規格資料
- **關聯鍵**: `primary_model`, `secondary_model`

**欄位 (Columns):**
- `secondary_model` - 次要型號
- `primary_model` - 主要型號
- `dmd_chip` - DMD 晶片
- `illumination_power` - 照明功率
- `light_source` - 光源
- `major_group` - 主要群組
- `min_brightness` - 最小亮度
- `minor_group` - 次要群組
- `native_resolution` - 原生解析度
- `owner` - 所有者
- `power_consumption_w_bright_220v` - 功耗 (亮模式, 220V)
- `power_consumption_w_eco_110v` - 功耗 (節能模式, 110V)
- `power_consumption_w_eco_220v` - 功耗 (節能模式, 220V)
- `pallet_quantity_air` - 棧板數量 (空運)
- `pallet_quantity_sea` - 棧板數量 (海運)
- `pixel_configuration` - 像素配置
- `pixe_pitch` - 像素間距
- `pixels` - 像素數
- `platform` - 平台
- `product_group` - 產品群組
- `projector_lens` - 投影鏡頭
- `scree_material` - 螢幕材質
- `screen_size` - 螢幕尺寸
- `screen_type` - 螢幕類型
- `sensor` - 感應器
- `supplier_group` - 供應商群組
- `supplier_tariff` - 供應商關稅
- `touch_type` - 觸控類型
- `typ_brightness` - 典型亮度
- `brightness` - 亮度
- `company_report_code_hq` - 公司報告代碼 (總部)

### Table Relationship
- 兩個表通過 `primary_model` 和 `secondary_model` 關聯
- `optw_dw_dsi_monthly_data` 記錄交易數據
- `optw_dw_dsi_model` 提供對應的型號規格資訊
