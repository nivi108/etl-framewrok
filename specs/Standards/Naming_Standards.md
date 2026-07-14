# Naming Standards

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** Project_Architecture.md (v1.0), Medallion_Architecture.md (v1.0), Repository_Structure.md (v1.0)
**Category:** Standards

## Purpose
Defines naming conventions for every named artifact in the framework — Delta tables, notebooks, workflows, config entries, and generated code files. Consistent naming is what makes metadata-driven automation possible: an agent resolving a table's Silver notebook name from config must produce a name that matches the notebook actually generated — if naming conventions vary between agents, the whole framework breaks at the seams.

## Scope
Covers naming patterns for artifacts. Does NOT cover variable/function naming inside code (that's `Coding_Standards.md`) and does NOT cover folder structure (that's `Repository_Structure.md`).

## General Rules

| Rule | Convention |
|---|---|
| Case | Always `lowercase_snake_case` — no camelCase, PascalCase, or kebab-case anywhere |
| Separators | Underscore `_` only — no hyphens, dots, or spaces in artifact names |
| Abbreviations | Avoid unless universally understood (`id`, `dt`, `ts`) — prefer `customer` over `cust`, `order` over `ord` |
| Prefixes | Layer prefixes are mandatory for tables and notebooks — ensures a name alone tells you which layer it belongs to |
| Reserved words | Never use SQL/PySpark reserved words as table or column names (`table`, `date`, `type`, `order`) — suffix with a descriptor instead (`order_date`, `load_type`) |

## Delta Table Naming

| Layer | Pattern | Example |
|---|---|---|
| Raw | `raw_{source_table}` | `raw_orders`, `raw_customers` |
| Silver | `silver_{source_table}` | `silver_orders`, `silver_customers` |
| Silver rejected | `silver_{source_table}_rejected` | `silver_orders_rejected` |
| Gold dimension | `dim_{entity}` | `dim_customer`, `dim_product` |
| Gold fact | `fact_{business_process}` | `fact_sales`, `fact_inventory_movement` |
| Gold aggregate | `agg_{summary_description}` | `agg_daily_sales`, `agg_monthly_revenue` |
| Config tables | `cfg_{purpose}` | `cfg_source`, `cfg_pipeline`, `cfg_workflow`, `cfg_validation` |
| Supporting tables | `reg_{purpose}` | `reg_schema`, `reg_watermark` |
| Log/audit tables | `log_{type}` or `audit_{scope}` | `log_execution`, `log_error`, `audit_reconciliation` |
| Profile tables | `profile_{scope}` | `profile_data` |

## Notebook Naming

| Layer | Pattern | Example |
|---|---|---|
| Raw ingestion | `load_{source_table}` | `load_orders`, `load_customers` |
| Silver cleansing | `clean_{source_table}` | `clean_orders`, `clean_customers` |
| Gold dimension | `build_dim_{entity}` | `build_dim_customer`, `build_dim_product` |
| Gold fact | `build_fact_{business_process}` | `build_fact_sales` |
| Gold aggregate | `build_agg_{summary}` | `build_agg_daily_sales` |
| Setup/init | `setup_{purpose}` | `setup_config_tables`, `setup_schema_registry` |
| Utility | `util_{purpose}` | `util_hash_computation`, `util_watermark_management` |

## Workflow Naming

| Pattern | Example |
|---|---|
| `wf_{domain}_{frequency}` | `wf_sales_daily`, `wf_inventory_hourly` |
| `wf_maintenance_{task}` | `wf_maintenance_optimize`, `wf_maintenance_vacuum` |
| `wf_setup_{scope}` | `wf_setup_initial_load` |

## Column Naming

| Category | Pattern | Example |
|---|---|---|
| Business columns | As-is from source, lowercased and snake_cased | `order_id`, `customer_name`, `order_date` |
| Audit columns (framework-added) | `_` prefix (underscore) to distinguish from source columns | `_ingested_at`, `_load_id`, `_source_system` |
| SCD columns | `_` prefix | `_effective_date`, `_expiry_date`, `_is_current`, `_version` |
| Hash columns | `_` prefix | `_row_hash` |
| Surrogate keys | `_surrogate_key` or `{entity}_key` in facts | `_surrogate_key` (in dim), `customer_key` (FK in fact) |

## Config Entry Naming

| Field | Convention |
|---|---|
| `table_name` in all config tables | Must exactly match the source table name, lowercased and snake_cased — this is the universal join key across all config tables |
| `workflow_name` | Must follow the workflow naming pattern above |
| `notebook_name` | Must follow the notebook naming pattern above |
| `dependency_group` | Short, descriptive, lowercase — matches the business domain: `sales`, `inventory`, `hr` |

## Validation Rules
- No generated artifact may deviate from these patterns — an agent producing `rawOrders` instead of `raw_orders` is a generation failure.
- Every name must be derivable from config fields + these patterns — if you know the `table_name` is `orders` and the layer is `Silver`, the table name is deterministically `silver_orders`, the notebook is `clean_orders`, and so on. No guesswork.

## Acceptance Criteria
- [ ] Every naming pattern produces a unique, deterministic name from config inputs alone.
- [ ] Layer prefixes (`raw_`, `silver_`, `dim_`, `fact_`, `agg_`) are consistently applied — no table exists without one.
- [ ] Framework-added columns are always `_` prefixed, never conflicting with source column names.
- [ ] No reserved words used as bare artifact names.

## Example (Illustrative — Full Naming Chain for One Table)
```
Source table: dbo.Orders

Config:
  table_name: orders

Generated artifacts:
  Raw table:        raw_orders
  Raw notebook:     load_orders
  Silver table:     silver_orders
  Silver rejected:  silver_orders_rejected
  Silver notebook:  clean_orders
  Gold fact:        fact_sales (named by business process, not source table)
  Gold notebook:    build_fact_sales
  Workflow:         wf_sales_daily
```

## Dependencies
- `Project_Architecture.md` (v1.0) — establishes the layers these prefixes map to.
- `Medallion_Architecture.md` (v1.0) — layer names (Raw/Silver/Gold) are the source of the prefix conventions.
- `Repository_Structure.md` (v1.0) — folder paths (`src/notebooks/raw/`, etc.) align with these naming patterns.

## Future Extension Points
- If a second source system is added, a source-system prefix could be introduced for Raw tables (e.g., `raw_sqlserver_orders` vs. `raw_salesforce_accounts`) to prevent name collisions.

## AI Generation Notes
Every agent must derive artifact names from config fields using these exact patterns — never invent names, never abbreviate beyond what's listed here, and never omit layer prefixes. If an agent needs to name an artifact type not covered here, that's a signal this spec needs extending before the agent proceeds.