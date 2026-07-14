# Full Load Component

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** Config_Framework.md (v1.0), Ingestion_Framework.md (v1.0), SQL_Server_Reader_Component.md (v1.0), Read_Config_Component.md (v1.0)
**Category:** Components

## Purpose
Implements the Full Load ingestion path defined in `Ingestion_Framework.md` — reading the entire source table on every run and writing it to Raw via overwrite. This is the simplest of the three load-type components, used for small tables or tables without a reliable incremental column.

## Scope
Covers only the Full Load path. Does NOT cover Incremental or CDC (separate sibling components) and does NOT cover the actual write mechanics beyond specifying the write mode (that's `Raw_Framework.md`'s responsibility).

## Inputs
| Input | Type | Description |
|---|---|---|
| `table_config` | object | Output of `Read_Config_Component`, for the table being loaded |

## Outputs
A Spark DataFrame containing the full current contents of the source table, validated and ready for the Raw write step.

## Configuration
No table-specific configuration beyond what's already in `Source_Config` (`load_type = Full` triggers this component's selection at the orchestration level — this component itself takes no additional parameters).

## Implementation Rules
- Always issues an unfiltered `SELECT *` (or explicit column list, if column selection is configured) against the source table — no watermark or incremental filtering logic present here.
- Always pairs with `mode=overwrite` at the Raw write step — this component must flag its output as `write_mode=overwrite` so `Raw_Framework.md`'s write logic applies the correct mode.
- Should warn (not fail) if the source table row count exceeds a configurable large-table threshold, since Full Load on very large tables is usually a sign the table should be reconfigured to Incremental or CDC instead.

## Error Handling
Delegates to `SQL_Server_Reader_Component.md` for connection/query-level errors. No additional error handling specific to Full Load beyond what that component already provides.

## Logging
Logs `table_name`, `load_type=Full`, row count read, and any large-table threshold warning.

## Return Values
Returns a validated Spark DataFrame ready for the Raw write step, or raises whatever exception `SQL_Server_Reader_Component` raises (propagated, not swallowed).

## Pseudo Logic
```
FUNCTION full_load(table_config):
    query = BUILD_QUERY(table_config.source_config, filter=None)
    df = SQL_Server_Reader_Component.read_from_sql_server(table_config.connection_config, query, fetch_mode="full")

    IF df.count() > LARGE_TABLE_THRESHOLD:
        LOG_WARNING("Full Load on large table - consider Incremental/CDC", table_config.table_name)

    RETURN {data: df, write_mode: "overwrite"}
```

## Acceptance Criteria
- [ ] No filtering logic present — always reads the complete table.
- [ ] Output is explicitly flagged for `overwrite` write mode.
- [ ] Large-table warning triggers without blocking execution.

## Example (Illustrative Only)
```
table_name: ProductCategories (small reference table)
load_type: Full
Result: entire table read every run, Raw table overwritten each time
```

## Dependencies
- `Ingestion_Framework.md` (v1.0) — this component implements the Full Load process flow defined there.
- `SQL_Server_Reader_Component.md` (v1.0) — handles the actual connection/query execution.
- `Read_Config_Component.md` (v1.0), `Config_Framework.md` (v1.0) — supplies `table_config`.

## Future Extension Points
- Could add column-pruning support (reading only configured columns rather than `SELECT *`) if source tables are wide and only a subset of columns are needed downstream.

## AI Generation Notes
Any agent generating a Full Load notebook must never introduce filtering logic here — if a table needs filtering, it should be reconfigured as Incremental or CDC in `Source_Config`, not patched with ad hoc filtering inside a Full Load notebook.