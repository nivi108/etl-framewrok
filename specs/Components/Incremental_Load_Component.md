# Incremental Load Component

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** Config_Framework.md (v1.0), Ingestion_Framework.md (v1.0), SQL_Server_Reader_Component.md (v1.0), Read_Config_Component.md (v1.0), Watermark_Component.md (v1.0)
**Category:** Components

## Purpose
Implements the Incremental Load path defined in `Ingestion_Framework.md` — reading only rows changed since the last successful watermark, using `incremental_column` and `watermark_column` from `Source_Config`.

## Scope
Covers only the Incremental Load path: watermark retrieval, filtered query construction, and handing off the result for writing. Does NOT advance the watermark itself (that happens only after a successful Raw write, per `Ingestion_Framework.md`'s recovery rules — advancement is triggered by `Raw_Framework.md`, not this component).

## Inputs
| Input | Type | Description |
|---|---|---|
| `table_config` | object | Output of `Read_Config_Component`, for the table being loaded |

## Outputs
A Spark DataFrame containing only rows where `incremental_column > last_watermark`, plus the watermark value that was used (to be advanced later, only after successful write).

## Configuration
Reads `incremental_column` and `watermark_column` from `table_config.source_config` — no additional configuration needed here.

## Implementation Rules
- Watermark value is retrieved via `Watermark_Component.md` before querying — never inferred from the data itself or hardcoded.
- Query filter is always `incremental_column > last_watermark` (strictly greater than) — using `>=` would risk re-reading the boundary row from the previous run.
- The watermark value used for this run must be captured and returned alongside the data, so it can be advanced later only if the write succeeds.
- If no rows are returned (no changes since last watermark), this is a valid, successful outcome — not an error or a warning.

## Error Handling
Delegates to `SQL_Server_Reader_Component.md` for connection/query-level errors. Additionally:
| Scenario | Behavior |
|---|---|
| `last_watermark` cannot be retrieved (first run for this table) | Use a configured default starting point (e.g., earliest date, or a `initial_load_date` field) rather than failing |
| `incremental_column` value type mismatch (e.g., expected timestamp, got string) | Raise immediately — this is a configuration/schema problem, not transient |

## Logging
Logs `table_name`, `load_type=Incremental`, `watermark_used`, row count read.

## Return Values
Returns `{data: DataFrame, watermark_used: value}`, or raises whatever exception the underlying reader/watermark component raises.

## Pseudo Logic
```
FUNCTION incremental_load(table_config):
    last_watermark = Watermark_Component.get_last_watermark(table_config.table_name)
    IF last_watermark IS NULL:
        last_watermark = table_config.source_config.initial_load_date  # first-run default

    query = BUILD_QUERY(
        table_config.source_config,
        filter=f"{table_config.source_config.incremental_column} > {last_watermark}"
    )
    df = SQL_Server_Reader_Component.read_from_sql_server(table_config.connection_config, query, fetch_mode="batch")

    LOG(table_name=table_config.table_name, watermark_used=last_watermark, row_count=df.count())
    RETURN {data: df, watermark_used=last_watermark}
    # Note: watermark is NOT advanced here - Raw_Framework.md advances it only after successful write
```

## Acceptance Criteria
- [ ] Never re-reads rows already captured in the previous successful run (strict `>` filter, not `>=`).
- [ ] Watermark advancement is explicitly deferred to the write step, never advanced speculatively within this component.
- [ ] Handles first-run (no prior watermark) gracefully via a configured default.

## Example (Illustrative Only)
```
table_name: Customers
incremental_column: modified_date
last_watermark: 2026-07-12T00:00:00
Query: SELECT * FROM dbo.Customers WHERE modified_date > '2026-07-12T00:00:00'
```

## Dependencies
- `Ingestion_Framework.md` (v1.0) — this component implements the Incremental Load process flow.
- `SQL_Server_Reader_Component.md` (v1.0) — connection/query execution.
- `Watermark_Component.md` (v1.0) — retrieves the last known watermark.
- `Read_Config_Component.md` (v1.0), `Config_Framework.md` (v1.0) — supplies `table_config`.

## Future Extension Points
- Could support composite watermarks (multiple columns) if a source table's change detection requires more than a single timestamp column.

## AI Generation Notes
Any agent generating an Incremental Load notebook must use strict `>` filtering on the watermark and must never advance the watermark within this component's own logic — advancement belongs exclusively to the Raw write step, per the idempotency guarantee in `Ingestion_Framework.md`.