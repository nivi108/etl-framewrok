# Skill: Create Silver Notebook

**Version:** 1.0
**Last Modified:** 2026-07-13
**Invoked By:** `silver_agent.md`, `orchestrator_agent.md`
**Category:** Skills

## Purpose
Step-by-step, mechanically repeatable recipe for generating one Silver cleansing notebook for a given table.

## Trigger
Called once per active table that has a Silver-layer `Pipeline_Config` row. Receives a `table_config` object.

## Pre-Conditions
- `table_config` includes a Silver-layer `Pipeline_Config` row with `merge_keys`, `business_keys`, `scd_type`.
- `Validation_Config` exists for this table with `dq_rules` and `error_handling_strategy`.

## Step-by-Step Generation

### Step 1: Resolve Names
```
notebook_name = f"clean_{table_config.source_config.table_name}"
output_path = f"src/notebooks/silver/{notebook_name}.py"
raw_table_name = f"raw_{table_config.source_config.table_name}"
silver_table_name = f"silver_{table_config.source_config.table_name}"
rejected_table_name = f"silver_{table_config.source_config.table_name}_rejected"
```

### Step 2: Write Notebook Header
```
- Import: pyspark.sql.functions as F
- Import: Read_Config_Component
- Import: Schema_Validation_Component
- Import: Null_Handling_Component
- Import: Type_Casting_Component
- Import: Dedup_Component
- Import: Hashing_Component (if scd_type == 2)
- Import: Merge_Component
- Import: Profiling_Component
- Import: Logging_Component
- Import: Audit_Component
- Parameter: environment (string)
- Parameter: table_name (string)
- Parameter: pipeline_id (string)
```

### Step 3: Write Main Function Skeleton
```
def main(environment, table_name, pipeline_id):
    Logging_Component.log_execution_start(...)
    try:
        config = Read_Config_Component.read_config(table_name)
        raw_df = spark.table(raw_table_name)  # read from Raw
        # Steps 4-11 go here
        Logging_Component.log_execution_end(..., status="Success", counts=...)
    except Exception as e:
        Logging_Component.log_error(...)
        Logging_Component.log_execution_end(..., status="Failed")
        raise
    finally:
        Audit_Component.write_audit(table_name, "Silver", pipeline_id, counts)
```

### Step 4: Write Stage 1 — Schema Validation
```
Logging_Component.log_notebook_step(..., "schema validation started")
schema_result = Schema_Validation_Component.validate_schema(table_name, raw_df.schema)
if schema_result.status == "Halt":
    raise SchemaValidationError(schema_result.diff)
```

### Step 5: Write Stage 2 — Null Handling
```
Logging_Component.log_notebook_step(..., "null handling started")
null_result = Null_Handling_Component.handle_nulls(raw_df, config.pipeline_config.null_rules)
accepted_df = null_result.accepted
rejected_nulls = null_result.rejected
```

### Step 6: Write Stage 3 — Type Casting
```
Logging_Component.log_notebook_step(..., "type casting started")
cast_result = Type_Casting_Component.cast_types(accepted_df, config.pipeline_config.column_type_map)
accepted_df = cast_result.accepted
rejected_casts = cast_result.rejected
```

### Step 7: Write Stage 4 — Deduplication
```
Logging_Component.log_notebook_step(..., "deduplication started")
dedup_result = Dedup_Component.deduplicate(
    accepted_df, config.pipeline_config.merge_keys,
    config.source_config.watermark_column, silver_table_ref
)
accepted_df = dedup_result.data
dedup_count = dedup_result.dedup_count
rejected_dedup = dedup_result.rejected
```

### Step 8: Write Stage 5 — Data Standardization
```
Logging_Component.log_notebook_step(..., "standardization started")
accepted_df = apply_standardization(accepted_df)
# trim whitespace, standardize casing, normalize date formats
```

### Step 9: Write Stages 6-7 — Business Rules + DQ Rules
```
Logging_Component.log_notebook_step(..., "business rule and DQ validation started")
for rule in config.validation_config.dq_rules:
    result = evaluate_rule(rule, accepted_df)
    if not result.passed:
        if rule.severity == "blocking":
            raise DQBlockingError(rule.rule_name)
        else:
            rejected_dq.append(result.failed_rows)
            accepted_df = accepted_df - result.failed_rows
```

### Step 10: Write Rejected Records Output
```
all_rejected = union(rejected_nulls, rejected_casts, rejected_dedup, rejected_dq)
all_rejected.write.format("delta").mode("append").saveAsTable(rejected_table_name)
```

### Step 11: Write Stage 8 — Merge into Silver
```
Logging_Component.log_notebook_step(..., "merge started")

if config.pipeline_config.scd_type == 2:
    accepted_df = Hashing_Component.add_row_hash(accepted_df, tracked_columns)
    Merge_Component.execute_merge(accepted_df, silver_table_name, merge_keys, merge_mode="scd2")
else:
    Merge_Component.execute_merge(accepted_df, silver_table_name, merge_keys, merge_mode="simple_upsert")
```

### Step 12: Write Profiling (Non-Blocking)
```
try:
    Profiling_Component.profile_batch(accepted_df, config, "Silver", pipeline_id)
except:
    pass  # non-blocking, per Profiling_Component rules
```

## Post-Conditions
- One `.py` file exists at `output_path`.
- All eight stages present in exact order.
- Rejected records routed to `{table_name}_rejected`, never dropped.

## Acceptance Criteria
- [ ] All eight Silver stages present and ordered correctly.
- [ ] SCD2 mode selected when `scd_type = 2`, simple_upsert otherwise.
- [ ] DQ rule severity read from config, not hardcoded.
- [ ] Profiling wrapped in non-blocking try/except.