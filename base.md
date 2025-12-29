database_config:
  engine: Redshift
  table_name: netsuite.optw_dw_dsi_st
  columns:
    region: "VARCHAR - Region code (e.g., AP)"
    primary_model: "VARCHAR - Main model name"
    secondary_model: "VARCHAR - Specific model name"
    dmd_chp: "DECIMAL - Demand parameter"
    section: "VARCHAR - Category (Sales, Purchase, Inventory)"
    data_type: "VARCHAR - Time period (YYYYMM) or Inventory type (FG, LII)"
    value: "DECIMAL - Numeric result"

business_logic:
  inventory_calculation:
    description: "Calculate Beginning Inventory"
    formula: "FG + In Transit"
    logic_type: "Inventory Cut-off"
    sql_filter: "SUM(CASE WHEN data_type = 'FG + In Transit' THEN value ELSE 0 END)"

  purchase_offset:
    description: "Purchase Forecast availability lag"
    rule: "M+1 (Purchase in current month is available next month)"
    implementation: "ADD_MONTHS(TO_DATE(LEFT(data_type, 6), 'YYYYMM'), 1)"
    trigger: "section = 'Purchase Forecast'"

  baseline_selection:
    description: "Selection of inventory starting point"
    rule: "Use previous month as the baseline for the current month"
    sorting_requirement: "Sort by date object, do not use string MAX()"

  query_rules:
    model_search:
      operator: "LIKE"
      target_column: "secondary_model"
      syntax: "WHERE secondary_model LIKE '%{model_name}%'"

section_mapping:
  actuals: "TOTAL SALES"
  forecast_demand: "Sales Forecast"
  forecast_supply: "Purchase Forecast"
  inventory_snapshot: "Inventory cut off date:*" # Use wildcard or regex
  delivery: "Delivery Plan"

inventory_types:
  - FG
  - FG + In Transit
  - LII
