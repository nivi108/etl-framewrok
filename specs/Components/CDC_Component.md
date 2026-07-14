# CDC Component

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** Config_Framework.md (v1.0), Ingestion_Framework.md (v1.0), SQL_Server_Reader_Component.md (v1.0), Read_Config_Component.md (v1.0), Watermark_Component.md (v1.0) [forward reference — not yet built]
**Category:** Components

## Purpose
Implements the CDC Load path defined in `Ingestion_Framework.md` — reading change data from SQL Server's CDC/Change Tracking mechanism since the last captured version, classifying each row by operation type (Insert/Update/Delete), and preparing it for Raw as soft-delete-preserving records.

## Scope
Covers CDC-specific read and classification logic only. Does NOT handle the Raw write mechanics (that's `Raw_Framework.md`) and does NOT advance the CDC version watermark itself (deferred to the write step, same principle as `Incremental_Load_Component.md`).

## ⚠ Forward Dependency Note
This component depends on version-tracking behavior equivalent to `Watermark_Component.md`, which has not yet been built in our sequence. CDC version/LSN tracking uses the same get/set pattern as a timestamp watermark, just with a different value type (version number/LSN string instead of a timestamp). Until `Watermark_Component.md` is built, treat this as a placeholder dependency with a predictable contract: `get_last_cdc_version(table_name)` / implicitly set on successful write.

## Inputs
| Input | Type | Description |
|---|---|---|
| `table_config` | object | Output of `Read_Config_Component`, for the table being loaded |

## Outputs
A Spark DataFrame containing changed rows since the last CDC version, each tagged with `_operation_type` (Insert/Update/Delete), plus the CDC version used for this run.

## Configuration
Reads `cdc_enabled` and `primary_keys` from `table_config.source_config`. Assumes SQL Server CDC or Change Tracking is already enabled at the source (this component does not enable CDC on the source — that's a one-time DBA-side setup, out of scope for this framework).

## Implementation Rules
- Every row returned must carry an `_operation_type` value — never left ambiguous.
- Delete operations are never physically excluded from the output — they are passed through with `_operation_type = Delete` so Raw can preserve them as soft-delete-flagged rows, per `Raw_Framework.md`.
- CDC version used for this run must be captured and returned alongside the data, for later advancement (only after successful write).
- If a table has multiple change rows for the same primary key within one capture window (e.g., an insert followed by an update in rapid succession), all changes are passed through — deduplication/collapsing to the latest state happens at Silver, not here.

## Error Handling
Delegates to `SQL_Server_Reader_Component.md` for connection/query-level errors. Additionally:
| Scenario | Behavior |
|---|---|
| CDC not actually enabled on source table despite `cdc_enabled = true` in config | Raise immediately — configuration/source mismatch, not transient, alert for manual correction |
| CDC change table has been cleaned up beyond the last captured version (retention gap on SQL Server side) | Raise a specific `CDCGapError` — this requires a Full Load reconciliation, not a normal retry |

## Logging
Logs `table_name`, `load_type=CDC`, `cdc_version_used`, row counts by operation type (Insert/Update/Delete separately).

## Return Values
Returns `{data: DataFrame, cdc_version_used: value}`, or raises the exceptions described above.

## Pseudo Logic
```
FUNCTION cdc_load(table_config):
    last_version = get_last_cdc_version(table_config.table_name)   # Watermark_Component equivalent

    TRY:
        changes = READ_CDC_CHANGES(table_config.connection_config, SINCE last_version)
    CATCH RetentionGapError:
        RAISE CDCGapError(table_config.table_name)

    classified = CLASSIFY_OPERATIONS(changes)   # tags _operation_type per row

    LOG(table_name=table_config.table_name, cdc_version_used=last_version,
        insert_count=classified.inserts, update_count=classified.updates, delete_count=classified.deletes)

    RETURN {data: classified, cdc_version_used: last_version}
    # CDC version is NOT advanced here - Raw_Framework.md advances it only after successful write
```

## Acceptance Criteria
- [ ] Every returned row has an explicit `_operation_type`.
- [ ] Deletes are passed through, never silently dropped.
- [ ] CDC version advancement is deferred to the write step, consistent with the Incremental Load pattern.
- [ ] A CDC retention gap is distinguished from a normal transient failure and handled differently.

## Example (Illustrative Only)
```
table_name: Orders
cdc_version_used: 000000A2:0000A1F0:0003
Result: 42 Insert, 15 Update, 3 Delete rows, all tagged with _operation_type
```

## Dependencies
- `Ingestion_Framework.md` (v1.0) — this component implements the CDC Load process flow.
- `SQL_Server_Reader_Component.md` (v1.0) — connection/query execution.
- `Watermark_Component.md` (not yet built) — version tracking, forward reference per note above.
- `Read_Config_Component.md` (v1.0), `Config_Framework.md` (v1.0) — supplies `table_config`.

## Future Extension Points
- Could add support for SQL Server Change Tracking as a lighter-weight alternative to full CDC, if some tables don't need full before/after change capture.

## AI Generation Notes
Any agent generating a CDC notebook must tag every row with `_operation_type` and must never silently drop delete records — Raw must always preserve the full change history for CDC-enabled tables.