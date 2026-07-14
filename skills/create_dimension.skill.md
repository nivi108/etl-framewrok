# Skill: Create Dimension Notebook

**Version:** 1.0
**Last Modified:** 2026-07-13
**Invoked By:** `gold_agent.md`, `orchestrator_agent.md`
**Category:** Skills

## Purpose
Step-by-step, mechanically repeatable recipe for generating one Gold dimension-building notebook for a given table.

## Trigger
Called once per table where `Pipeline_Config.fact_or_dimension = Dimension` at the Gold layer. Receives a `table_config` object.

## Pre-Conditions
- `table_config` has a Gold-layer `Pipeline_Config` row with `fact_or_dimension = Dimension`.
- `business_keys` and `merge_keys` are populated.

## Step-by-Step Generation

### Step 1: Resolve Names
```
entity_name = derive_entity_name(table_config)  # e.g., "customer" from silver_customers
notebook_name = f"build_dim_{entity_name}"
output_path = f"src/notebooks/gold/{notebook_name}.py"
silver_table_name = f"silver_{table_config.source_config.table_name}"
dim_table_name = f"dim_{entity_name}"
```

### Step 2: Write Notebook Header
```
- Import: pyspark.sql.functions as F
- Import: Read_Config_Component
- Import: Surrogate_Key_Component
- Import: Hashing_Component (if scd_type == 2)
- Import: Merge_Component
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
        silver_df = spark.table(silver_table_name)
        # Steps 4-7 go here
        Logging_Component.log_execution_end(..., status="Success", counts=...)
    except Exception as e:
        Logging_Component.log_error(...)
        Logging_Component.log_execution_end(..., status="Failed")
        raise
    finally:
        Audit_Component.write_audit(table_name, "Gold", pipeline_id, counts)
```

### Step 4: Write Unknown Member Guarantee
```
Logging_Component.log_notebook_step(..., "ensuring Unknown member row exists")
ensure_unknown_member(dim_table_name, config.pipeline_config.business_keys)
# Idempotent: checks existence, creates only if missing
# Unknown member surrogate_key = -1, all attributes set to "Unknown"
```

### Step 5: Write Surrogate Key Resolution
```
Logging_Component.log_notebook_step(..., "resolving surrogate keys")
keyed_df = silver_df.map(lambda row:
    row + Surrogate_Key_Component.resolve_surrogate_key(
        row[business_key_column], dim_table_name
    )
)
```

### Step 6: Write Hash Computation (SCD2 Only)
```
if config.pipeline_config.scd_type == 2:
    Logging_Component.log_notebook_step(..., "computing row hashes for SCD2")
    keyed_df = Hashing_Component.add_row_hash(keyed_df, tracked_columns)
```

### Step 7: Write Merge
```
Logging_Component.log_notebook_step(..., "merging into dimension")
merge_mode = "scd2" if config.pipeline_config.scd_type == 2 else "simple_upsert"
result = Merge_Component.execute_merge(
    keyed_df, dim_table_name, config.pipeline_config.merge_keys, merge_mode
)
```

## Post-Conditions
- One `.py` file exists at `output_path`.
- Unknown member row guaranteed before any merge.
- Surrogate keys resolved via shared component, not ad hoc.

## Acceptance Criteria
- [ ] Unknown member creation is idempotent and runs before merge.
- [ ] Surrogate key resolution uses `resolve_surrogate_key` (generate-or-lookup), not `lookup_only`.
- [ ] SCD2 hash computation present when `scd_type = 2`, absent otherwise.
- [ ] Merge mode matches `scd_type` configuration.