# Skill: Create Raw Notebook

**Version:** 1.0
**Last Modified:** 2026-07-13
**Invoked By:** `raw_agent.md`, `orchestrator_agent.md`
**Category:** Skills

## Purpose
Step-by-step, mechanically repeatable recipe for generating one Raw ingestion notebook for a given table. An AI model following these steps should produce identical output for identical inputs, with zero interpretation needed.

## Trigger
Called once per active table in `Source_Config`. Receives a `table_config` object.

## Pre-Conditions
- `table_config` is valid (returned by `Read_Config_Component`, not null, `active_flag = true`).
- `Source_Config.load_type` is one of: `Full`, `Incremental`, `CDC`.

## Step-by-Step Generation

### Step 1: Resolve Names
```
notebook_name = f"load_{table_config.source_config.table_name}"
output_path = f"src/notebooks/raw/{notebook_name}.py"
raw_table_name = f"raw_{table_config.source_config.table_name}"
```

### Step 2: Write Notebook Header
Generate the imports and parameter block:
```
- Import: pyspark.sql.functions as F
- Import: Read_Config_Component
- Import: SQL_Server_Reader_Component
- Import: Schema_Validation_Component
- Import: Watermark_Component (if Incremental or CDC)
- Import: Partitioning_Component
- Import: Logging_Component
- Import: Audit_Component
- Parameter: environment (string, no default)
- Parameter: table_name (string, no default)
- Parameter: pipeline_id (string, no default)
```

### Step 3: Write Main Function Skeleton
```
def main(environment, table_name, pipeline_id):
    Logging_Component.log_execution_start(pipeline_id, ..., table_name)
    try:
        config = Read_Config_Component.read_config(table_name)
        # Steps 4-8 go here
        Logging_Component.log_execution_end(pipeline_id, status="Success", counts=...)
    except Exception as e:
        Logging_Component.log_error(pipeline_id, ..., str(e), traceback)
        Logging_Component.log_execution_end(pipeline_id, status="Failed", counts=...)
        raise
    finally:
        Audit_Component.write_audit(table_name, "Raw", pipeline_id, counts, watermark_state)
```

### Step 4: Write Source Read Block
Branch on `load_type`:

**If Full:**
```
query = f"SELECT * FROM {config.source_config.schema_name}.{config.source_config.table_name}"
source_df = SQL_Server_Reader_Component.read_from_sql_server(connection_config, query, fetch_mode="full")
write_mode = "overwrite"
```

**If Incremental:**
```
last_watermark = Watermark_Component.get_last_watermark(table_name, "timestamp")
query = f"SELECT * FROM ... WHERE {config.source_config.incremental_column} > '{last_watermark}'"
source_df = SQL_Server_Reader_Component.read_from_sql_server(connection_config, query, fetch_mode="batch")
write_mode = "append"
```

**If CDC:**
```
last_version = Watermark_Component.get_last_watermark(table_name, "cdc_version")
source_df = SQL_Server_Reader_Component.read_cdc_changes(connection_config, since=last_version)
write_mode = "merge"
```

### Step 5: Write Schema Validation Block
```
schema_result = Schema_Validation_Component.validate_schema(table_name, source_df.schema)
if schema_result.status == "Halt":
    raise SchemaValidationError(schema_result.diff)
```

### Step 6: Write Audit Column Addition
```
source_df = source_df.withColumn("_ingested_at", F.current_timestamp())
source_df = source_df.withColumn("_source_system", F.lit(config.source_config.source_system))
source_df = source_df.withColumn("_load_id", F.lit(load_id))
source_df = source_df.withColumn("_watermark_value", F.lit(str(watermark_used)))
# If CDC: source_df already has _operation_type from the reader
```

### Step 7: Write Duplicate Check (Incremental/CDC Only)
```
if write_mode != "overwrite":
    existing_load = spark.table(raw_table_name).filter(F.col("_load_id") == load_id)
    if existing_load.count() > 0:
        # Already written, skip to watermark advancement
        skip_write = True
```

### Step 8: Write Raw Table Write Block
```
if not skip_write:
    partitioned_writer = Partitioning_Component.apply_partitioning(
        source_df, config.pipeline_config.partition_columns, "write"
    )
    partitioned_writer.format("delta").mode(write_mode).saveAsTable(raw_table_name)
```

### Step 9: Write Watermark Advancement (Incremental/CDC Only)
```
if load_type != "Full":
    Watermark_Component.advance_watermark(table_name, new_watermark_value, load_id)
```

## Post-Conditions
- One `.py` file exists at `output_path`.
- Notebook follows `Coding_Standards.md` (type hints, docstrings, no magic numbers).
- All shared Components called, no inline reimplementations.

## Acceptance Criteria
- [ ] Generated notebook compiles without syntax errors.
- [ ] Load type branching matches `Source_Config.load_type` exactly.
- [ ] Logging at start/end, audit in finally block.
- [ ] Watermark advanced only after successful write.