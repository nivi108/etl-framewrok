# Partitioning Component

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** Config_Framework.md (v1.0), Raw_Framework.md (v1.0), Gold_Framework.md (v1.0)
**Category:** Components

## Purpose
Implements Delta table partitioning across Raw, Silver, and Gold layers — applying the partition columns declared in `Pipeline_Config.partition_columns`, and providing consistent partition-management helpers (partition pruning targets, partition-scoped operations). This is the single component all layers call for partition-related concerns.

## Scope
Covers partition specification and partition-scoped operation helpers only. Does NOT decide *which* columns to partition on (that's per-table config, with defaults recommended in `Raw_Framework.md` and `Gold_Framework.md`) and does NOT perform table maintenance operations like `OPTIMIZE`/`VACUUM` directly (those are separate maintenance jobs referenced in `Raw_Framework.md`).

## Inputs
| Input | Type | Description |
|---|---|---|
| `partition_columns` | array[string] | From `Pipeline_Config.partition_columns` |
| `write_operation` | enum(create, write, target_partitions) | What partition-related action is needed |

## Outputs
Depends on the operation:
- `create` — Delta table creation options with partitioning applied.
- `write` — DataFrame writer options with partition specification.
- `target_partitions` — a list of partition values that a given batch will write to (used for partition pruning and incremental aggregation).

## Configuration
`partition_columns` sourced from `Pipeline_Config`. Optional framework-wide defaults may exist (e.g., "if not specified, use `_ingested_at` for Raw"), but any actual override always comes from config, never from this component's own logic.

## Implementation Rules
- Never partition on a high-cardinality column (e.g., primary key, order_id) — this creates thousands of tiny partitions and destroys performance. This is enforced as a validation rule during config review, not a runtime check here, but should be flagged if detected.
- Partition column values must never be null — a null partition value fragments data unpredictably. If a null appears in a partition column, route the row to rejected records rather than writing it into a nameless partition.
- Partition columns should be low-cardinality and query-relevant (e.g., date, region, source_system) — the ideal partition strategy makes common query filters partition-prunable.

## Error Handling
| Scenario | Behavior |
|---|---|
| `partition_columns` refers to a column that doesn't exist in the DataFrame | Raise immediately — this is a config/schema mismatch, not a data issue |
| A row has a null value in a partition column | Reject the row, annotate `_rejection_reason` with the null partition column name |
| `partition_columns` is empty/null | Valid — the table simply won't be partitioned; not an error |

## Logging
Logs partition column list applied per operation, count of partitions written to per batch, count of rows rejected due to null partition values.

## Return Values
Returns the appropriate object per operation (writer options, table creation spec, or partition list) — or raises on invalid config.

## Pseudo Logic
```
FUNCTION apply_partitioning(dataframe, partition_columns, write_operation):
    IF partition_columns IS EMPTY:
        RETURN dataframe.writer   # no partitioning applied

    FOR each col in partition_columns:
        IF col NOT IN dataframe.columns:
            RAISE ConfigError(f"partition column {col} not in dataframe")

    null_partition_rows = FILTER(dataframe, ANY partition_columns IS NULL)
    valid_rows = dataframe - null_partition_rows

    IF write_operation == "create":
        RETURN CREATE_TABLE_OPTIONS(partition_by=partition_columns)
    ELIF write_operation == "write":
        RETURN valid_rows.writer.partitionBy(partition_columns)
    ELIF write_operation == "target_partitions":
        RETURN DISTINCT(valid_rows, partition_columns)

    LOG(partition_columns, rejected_null_count=null_partition_rows.count())
```

## Acceptance Criteria
- [ ] Partition column selection is always driven by config, never hardcoded per table.
- [ ] Null partition values result in row rejection, never writes to a null-named partition.
- [ ] Missing/empty `partition_columns` config is a valid state, not an error.

## Example (Illustrative Only)
```
partition_columns: [_ingested_at]
Result: raw_orders table partitioned by _ingested_at (date); each daily load writes to one new partition
```

## Dependencies
- `Raw_Framework.md` (v1.0) — Raw partitioning defaults documented there.
- `Gold_Framework.md` (v1.0) — Gold fact partitioning by date/time grain.
- `Config_Framework.md` (v1.0) — `partition_columns` sourced from `Pipeline_Config`.

## Future Extension Points
- Could support generated partition columns (e.g., derive `year_month` from a `date` column automatically for coarser partitioning) if fine-grained date partitioning proves excessive for some tables.
- Could integrate with Delta's liquid clustering as a future replacement/complement to explicit partitioning, if adopted framework-wide.

## AI Generation Notes
Any agent generating layer-processing notebooks must call this shared component for all partition specifications — never inline `.partitionBy(...)` calls per notebook, since that bypasses the null-partition-rejection safety and centralized config-driven logic.