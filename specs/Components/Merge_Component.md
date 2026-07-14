# Merge Component

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** Config_Framework.md (v1.0), Silver_Framework.md (v1.0), SCD_Type2_Framework.md (v1.0), Gold_Framework.md (v1.0)
**Category:** Components

## Purpose
Implements the shared Delta MERGE/upsert logic used both by Silver (stage 8, including SCD Type 2 merges) and Gold (dimension merges). Rather than each layer writing its own merge statement, all merge operations across the framework funnel through this single component, parameterized by merge keys and mode.

## Scope
Covers the mechanics of executing a Delta MERGE operation. Does NOT decide business logic about what should be merged (that's determined by the calling Framework — `Silver_Framework.md`, `SCD_Type2_Framework.md`, or `Gold_Framework.md`) — this component is purely the execution mechanism.

## Inputs
| Input | Type | Description |
|---|---|---|
| `source_data` | DataFrame | The incoming data to merge |
| `target_table` | string | The Delta table to merge into |
| `merge_keys` | array[string] | Columns identifying a unique row for match conditions |
| `merge_mode` | enum(simple_upsert, scd2) | Whether this is a plain upsert or an SCD2-aware merge (expire + insert) |
| `update_columns` | array[string], optional | For `simple_upsert`: which columns to update on match (defaults to all non-key columns if omitted) |

## Outputs
Merge execution result: `{rows_inserted: int, rows_updated: int, rows_deleted: int (if applicable)}` — used directly by `Audit_Framework.md`'s reconciliation counts.

## Configuration
No independent configuration — driven entirely by the caller's `merge_keys` (from `Pipeline_Config`) and `merge_mode` (determined by the caller based on `scd_type`).

## Implementation Rules
- `simple_upsert` mode: standard `MERGE INTO target USING source ON target.key = source.key WHEN MATCHED THEN UPDATE ... WHEN NOT MATCHED THEN INSERT ...`.
- `scd2` mode: a two-part operation — (1) expire currently-active rows whose business key appears in the source with a changed hash (per `Hashing_Component.md`/`SCD_Type2_Framework.md` logic), (2) insert new version rows. This must be executed as a single atomic transaction where possible (Delta's MERGE with a CASE-based update+insert, or two operations wrapped to behave atomically from the caller's perspective).
- Every merge operation must be idempotent — re-running the exact same merge with the exact same source data must not change the row counts a second time (this is what makes retries safe, per `Ingestion_Framework.md`'s and `Raw_Framework.md`'s recovery principles extended to Silver/Gold).

## Error Handling
| Scenario | Behavior |
|---|---|
| Merge key collision (multiple source rows match the same target row ambiguously) | Raise immediately — Delta MERGE itself will error on this, and it should not be caught/suppressed, since it indicates a dedup failure upstream |
| Target table doesn't exist yet (first-ever load) | Create the table via the merge operation, or via a separate first-load path — determined by which layer is calling (documented in that layer's Framework) |

## Logging
Logs `target_table`, `merge_mode`, rows inserted/updated/deleted, and execution duration. Feeds directly into `Logging_Framework.md`'s row-count fields.

## Return Values
Returns the merge execution result object, or propagates the underlying Delta MERGE exception if one occurs (e.g., key collision).

## Pseudo Logic
```
FUNCTION execute_merge(source_data, target_table, merge_keys, merge_mode, update_columns=None):
    IF merge_mode == "simple_upsert":
        RESULT = DELTA_MERGE(
            target=target_table,
            source=source_data,
            match_condition=EQUALS(merge_keys),
            on_match=UPDATE(update_columns OR all_non_key_columns),
            on_no_match=INSERT_ALL
        )

    ELIF merge_mode == "scd2":
        changed_keys = IDENTIFY_CHANGED(source_data, target_table, merge_keys)  # via Hashing_Component
        EXPIRE(target_table, WHERE business_key IN changed_keys AND _is_current = true)
        new_versions = BUILD_NEW_VERSIONS(source_data, changed_keys)
        RESULT = INSERT(target_table, new_versions)

    LOG(target_table, merge_mode, RESULT.rows_inserted, RESULT.rows_updated)
    RETURN RESULT
```

## Acceptance Criteria
- [ ] Merge operations are idempotent — re-running with identical source data produces zero additional changes.
- [ ] SCD2 mode correctly performs expire+insert as a coherent operation, never leaving a business key with zero or multiple current rows mid-operation.
- [ ] Merge key collisions are never silently suppressed.

## Example (Illustrative Only)
```
merge_mode: scd2
target_table: silver_customers
merge_keys: [customer_id]
Result: 3 rows expired, 3 new version rows inserted, 0 updated (SCD2 never updates in place)
```

## Dependencies
- `Silver_Framework.md` (v1.0), `SCD_Type2_Framework.md` (v1.0) — Silver merge stage (simple and SCD2 modes).
- `Gold_Framework.md` (v1.0) — Gold dimension merges use this same component.
- `Config_Framework.md` (v1.0) — `merge_keys` sourced from `Pipeline_Config`.

## Future Extension Points
- Could add a `soft_delete_merge` mode if source deletes need explicit handling beyond what CDC's operation-type tagging already provides at Raw.

## AI Generation Notes
Any agent generating merge logic for Silver or Gold must call this shared component rather than writing bespoke MERGE statements per table — this is precisely the kind of logic that must stay centralized for the framework to remain maintainable across "hundreds of projects."