# Config Framework

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** Project_Architecture.md (v1.0), Medallion_Architecture.md (v1.0), Repository_Structure.md (v1.0)
**Category:** Frameworks

## Purpose
Defines the complete metadata model that drives the entire ETL framework at runtime. This is the single most depended-upon Framework document — almost every other Framework and Component spec reads fields defined here. It answers: what metadata exists, what each field means, and which config table it lives in.

## Scope
Covers the *structure* of runtime configuration (field names, meaning, data types, which table they belong to). Does NOT cover actual row values for real tables (that's runtime data, entered later during onboarding) and does NOT cover schema *validation behavior* (that's `Schema_Management_Framework.md`).

## Why This Exists
Per our earlier discussion: this framework is what makes onboarding a new table a "config-only" exercise rather than a spec-rewriting exercise. Every Framework and Component downstream reads its instructions from these fields, not from hardcoded logic.

## Config Tables Overview

| Config Table | Purpose |
|---|---|
| `Source_Config` | Identifies source system, connection details, and ingestion method per table |
| `Pipeline_Config` | Defines layer-specific behavior — keys, SCD type, transformation notebook, target layer |
| `Workflow_Config` | Defines orchestration — schedule, dependency group, execution order, retry |
| `Validation_Config` | Defines DQ rules, validation rules, and error handling strategy per table |
| `schema_registry` | Supporting table storing last-known schema snapshot per table for drift detection. Used by `Schema_Management_Framework.md`. |
| `watermark_registry` | Supporting table storing last-processed watermark/CDC version per table. Used by `Watermark_Component.md`. |

These four core tables together form the complete metadata model. A table is only ready to be processed once it has a corresponding, active row in all four. `schema_registry` and `watermark_registry` are supporting tables used for drift detection and load tracking respectively.

## Field Definitions

### Source_Config
| Field | Type | Description |
|---|---|---|
| `source_system` | string | Name of the source system (e.g., "SQLServer_Prod") |
| `database_name` | string | Source database name |
| `schema_name` | string | Source schema name |
| `table_name` | string | Source table name |
| `load_type` | enum(Full, Incremental, CDC) | Ingestion method (see `Ingestion_Framework.md`) |
| `incremental_column` | string, nullable | Column used to detect new/changed rows (required if `load_type = Incremental`) |
| `watermark_column` | string, nullable | Column used to track last successfully loaded value |
| `cdc_enabled` | boolean | Whether CDC is enabled on the source table |
| `primary_keys` | array[string] | Source-side primary key column(s) |
| `active_flag` | boolean | Whether this table is currently eligible for processing |

### Pipeline_Config
| Field | Type | Description |
|---|---|---|
| `table_name` | string | Matches `Source_Config.table_name` |
| `target_layer` | enum(Raw, Silver, Gold) | Which layer this config row's rules apply to (one row per layer per table, or a combined row — implementation choice, but field must exist) |
| `merge_keys` | array[string] | Keys used in Silver/Gold MERGE operations |
| `business_keys` | array[string] | Business-meaningful keys, distinct from surrogate keys |
| `scd_type` | enum(1, 2, None) | SCD handling strategy for Silver/Gold |
| `fact_or_dimension` | enum(Fact, Dimension, None) | Applicable only for Gold layer rows |
| `partition_columns` | array[string] | Columns used for Delta partitioning |
| `transformation_notebook` | string | Name of the notebook responsible for this table's transformation logic at this layer |
| `retention_policy` | string | e.g., "90_days", "indefinite" |
| `measure_definitions` | array[object], optional | For Gold Fact tables only. Each object: `{measure_name: string, expression: string}` — e.g., `{measure_name: "line_total", expression: "quantity * unit_price"}`. Defined in detail in `Gold_Framework.md`. |
| `fact_write_mode` | enum(append, merge), optional | For Gold Fact tables only. Whether facts are appended (immutable events) or merged (correctable facts). Defined in `Gold_Framework.md`. |
| `null_rules` | array[object], optional | For Silver-layer rows. Per-column null handling. Each object: `{column_name: string, nullable: boolean, default_value: string (optional)}`. Defined in detail in `Null_Handling_Component.md`. |
| `column_type_map` | array[object], optional | For Silver-layer rows. Per-column type casting targets. Each object: `{column_name: string, target_data_type: string}`. Defined in detail in `Type_Casting_Component.md`. |
| `dimension_refs` | array[object], optional | For Gold Fact tables only. Which dimensions this fact references. Each object: `{dimension_table: string, source_column: string, foreign_key_column: string}`. Defined in detail in `Fact_Component.md`. |
| `aggregate_definition` | object, optional | For Gold Aggregate tables only. `{group_by: array[string], aggregations: array[{measure_name: string, source_column: string, function: enum(sum, count, avg, min, max, count_distinct)}], refresh_mode: enum(incremental, full_recompute)}`. Defined in detail in `Aggregation_Component.md`. |

### Workflow_Config
| Field | Type | Description |
|---|---|---|
| `table_name` | string | Matches `Source_Config.table_name` |
| `workflow_name` | string | Name of the Databricks Workflow this table belongs to |
| `notebook_name` | string | Entry-point notebook for this table's pipeline |
| `schedule` | string (cron) | Execution schedule |
| `dependency_group` | string | Logical grouping used to sequence related tables (e.g., "Sales") |
| `execution_order` | integer | Order within its dependency group |
| `parallel_group` | integer | Tables sharing this value may run in parallel |
| `execution_sequence` | integer | Global sequence number if strict ordering is required across groups |
| `retry_count` | integer | Number of retries on failure |
| `timeout_minutes` | integer | Max execution time before the job is killed |

### Validation_Config
| Field | Type | Description |
|---|---|---|
| `table_name` | string | Matches `Source_Config.table_name` |
| `validation_rules` | array[string] | References to named rules (e.g., "not_null_order_id") defined in `Data_Quality_Framework.md` |
| `dq_rules` | array[object] | Data quality checks specific to this table. Each object: `{rule_name: string, rule_type: enum, severity: enum(blocking, non_blocking), target_column: string (optional), expression: string, parameters: object (optional)}`. Full structure defined in `Data_Quality_Framework.md`. |
| `error_handling_strategy` | enum(Halt, SkipRow, SkipTable, Alert) | What happens when validation fails — see `Error_Handling_Framework.md` |

## Relationship Between Tables (Decision Table)

| Relationship | Rule |
|---|---|
| `Source_Config` → `Pipeline_Config` | One-to-many: one source table can have multiple Pipeline_Config rows (one per target layer) |
| `Source_Config` → `Workflow_Config` | One-to-one per table (each table belongs to exactly one workflow entry point) |
| `Source_Config` → `Validation_Config` | One-to-many: a table may have multiple validation rules attached |
| `active_flag = false` | Table is skipped entirely, regardless of what other config says |

## Example Metadata (Illustrative Only)

```yaml
Source_Config:
  source_system: SQLServer_Prod
  database_name: SalesDB
  schema_name: dbo
  table_name: Orders
  load_type: CDC
  watermark_column: modified_date
  cdc_enabled: true
  primary_keys: [order_id]
  active_flag: true

Pipeline_Config (Silver row):
  table_name: Orders
  target_layer: Silver
  merge_keys: [order_id]
  business_keys: [order_id]
  scd_type: 2
  transformation_notebook: clean_orders
  retention_policy: indefinite

Workflow_Config:
  table_name: Orders
  workflow_name: Sales_Pipeline
  notebook_name: load_orders
  schedule: "0 2 * * *"
  dependency_group: Sales
  execution_order: 1
  parallel_group: 2
  retry_count: 3
  timeout_minutes: 45

Validation_Config:
  table_name: Orders
  validation_rules: [not_null_order_id]
  dq_rules:
    - rule_name: positive_amount_check
      rule_type: range_check
      severity: blocking
      target_column: amount
      expression: "amount > 0"
  error_handling_strategy: Alert
```

## Best Practices
- Never let a Framework or Component doc invent a new config field inline — if a new field is needed, this document must be updated first, and its dependents flagged per `Spec_Versioning.md`.
- Keep field names consistent across all four tables (e.g., always `table_name`, never `tbl_name` in one place and `table` in another) — this is what allows agents to join across config tables reliably.

## Validation Rules
- Every `table_name` referenced in `Pipeline_Config`, `Workflow_Config`, or `Validation_Config` must exist in `Source_Config`.
- `incremental_column` must not be null when `load_type = Incremental`.
- `scd_type` must not be `None` if `target_layer = Silver` and history tracking is required by business rules (flagged as a manual review point during onboarding, not auto-enforced).

## Pseudo Logic
```
FUNCTION get_active_pipeline_config():
    tables = SELECT * FROM Source_Config WHERE active_flag = true
    FOR each table in tables:
        pipeline = JOIN Pipeline_Config ON table_name
        workflow = JOIN Workflow_Config ON table_name
        validation = JOIN Validation_Config ON table_name
        YIELD combined_config(table, pipeline, workflow, validation)
```

## Acceptance Criteria
- [ ] All four config tables' fields are fully documented with type and description.
- [ ] Every field referenced by any downstream Framework/Component doc traces back to a field defined here.
- [ ] Relationships between tables are unambiguous (one-to-one vs one-to-many clearly stated).

## Dependencies
- `Project_Architecture.md` (v1.0) — establishes config tables as the runtime metadata source.
- `Medallion_Architecture.md` (v1.0) — `target_layer` and `scd_type` fields map directly to layer contracts defined there.
- `Repository_Structure.md` (v1.0) — establishes the repo layout where generated DDL for these tables lives (`src/sql/ddl/`).

## Future Extension Points
- Adding a new source system type would likely require new fields in `Source_Config` (e.g., `api_endpoint` for an API source) — this doc should be revised first before any Component spec assumes those fields exist.
- A fifth config table could be added later (e.g., `Notification_Config`) if alerting logic grows complex enough to warrant separation from `Validation_Config`.

## AI Generation Notes
Any agent generating DDL for these config tables, or generating code that reads them, must use the exact field names and types defined above — no renaming, abbreviating, or restructuring without this spec being updated first.