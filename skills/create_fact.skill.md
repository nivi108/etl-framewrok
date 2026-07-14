# Skill: Create Fact Notebook

**Version:** 1.0
**Last Modified:** 2026-07-13
**Invoked By:** `gold_agent.md`, `orchestrator_agent.md`
**Category:** Skills

## Purpose
Step-by-step, mechanically repeatable recipe for generating one Gold fact-building notebook for a given table.

## Trigger
Called once per table where `Pipeline_Config.fact_or_dimension = Fact` at the Gold layer. Receives a `table_config` object including `dimension_refs` and `measure_definitions`.

## Pre-Conditions
- `table_config` has a Gold-layer `Pipeline_Config` row with `fact_or_dimension = Fact`.
- All referenced dimensions have already been built in this run (enforced by orchestrator/workflow ordering).
- `dimension_refs`, `measure_definitions`, and `fact_write_mode` are populated.

## Step-by-Step Generation

### Step 1: Resolve Names
```
business_process = derive_business_process(table_config)  # e.g., "sales"
notebook_name = f"build_fact_{business_process}"
output_path = f"src/notebooks/gold/{notebook_name}.py"
silver_table_name = f"silver_{table_config.source_config.table_name}"
fact_table_name = f"fact_{business_process}"
```

### Step 2: Write Notebook Header
```
- Import: pyspark.sql.functions as F
- Import: Read_Config_Component
- Import: Surrogate_Key_Component
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

### Step 4: Write Dimension Key Resolution
```
Logging_Component.log_notebook_step(..., "resolving dimension foreign keys")
resolved_df = silver_df
unknown_member_counts = {}

for dim_ref in config.pipeline_config.dimension_refs:
    resolved_df = resolved_df.map(lambda row:
        surrogate = Surrogate_Key_Component.lookup_only(
            row[dim_ref.source_column], dim_ref.dimension_table
        )
        if surrogate is None:
            surrogate = UNKNOWN_MEMBER_KEY  # -1
            unknown_member_counts[dim_ref.dimension_table] += 1
        row[dim_ref.foreign_key_column] = surrogate
    )

if any(count > 0 for count in unknown_member_counts.values()):
    Logging_Component.log_notebook_step(..., f"unresolved keys: {unknown_member_counts}")
```

### Step 5: Write Measure Computation
```
Logging_Component.log_notebook_step(..., "computing measures")
for measure in config.pipeline_config.measure_definitions:
    resolved_df = resolved_df.withColumn(
        measure.measure_name, F.expr(measure.expression)
    )
```

### Step 6: Write Null FK Validation
```
for dim_ref in config.pipeline_config.dimension_refs:
    null_fk_count = resolved_df.filter(F.col(dim_ref.foreign_key_column).isNull()).count()
    if null_fk_count > 0:
        raise DataIntegrityError(f"null FK in {dim_ref.foreign_key_column} after resolution")
```

### Step 7: Write Fact Table Write
```
Logging_Component.log_notebook_step(..., "writing to fact table")
if config.pipeline_config.fact_write_mode == "merge":
    Merge_Component.execute_merge(
        resolved_df, fact_table_name, config.pipeline_config.merge_keys, "simple_upsert"
    )
else:  # append
    Partitioning_Component.apply_partitioning(
        resolved_df, config.pipeline_config.partition_columns, "write"
    ).format("delta").mode("append").saveAsTable(fact_table_name)
```

## Post-Conditions
- One `.py` file exists at `output_path`.
- All dimension FKs resolved (real or Unknown member), never null.
- Measures computed from declarative expressions.

## Acceptance Criteria
- [ ] Uses `lookup_only` for dimension key resolution, never `resolve_surrogate_key`.
- [ ] Falls back to Unknown member key, never null FK.
- [ ] Post-resolution null FK check raises immediately if any null slips through.
- [ ] Measures use `F.expr()` evaluation, not hardcoded per-table logic.