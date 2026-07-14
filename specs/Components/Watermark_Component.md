# Watermark Component

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** Config_Framework.md (v1.0), Ingestion_Framework.md (v1.0), Audit_Framework.md (v1.0)
**Category:** Components

## Purpose
Manages retrieval and advancement of watermark values (for Incremental loads) and CDC version/LSN values (for CDC loads). This is the component `Incremental_Load_Component.md` and `CDC_Component.md` both forward-referenced — this file closes those gaps.

## Scope
Covers get/set operations for both watermark types (timestamp-based and CDC version-based) under a single unified interface. Does NOT decide when advancement happens (that's the Raw write step's responsibility, per `Ingestion_Framework.md`'s rule: advance only after successful write) — this component only provides the mechanism, not the trigger.

## Inputs (get_last_watermark)
| Input | Type | Description |
|---|---|---|
| `table_name` | string | Table to retrieve the watermark for |
| `watermark_type` | enum(timestamp, cdc_version) | Which type of watermark to retrieve |

## Inputs (advance_watermark)
| Input | Type | Description |
|---|---|---|
| `table_name` | string | Table to update |
| `new_value` | string/timestamp | The new watermark/CDC version value to persist |
| `load_id` | string | The `_load_id` this advancement is associated with, for traceability |

## Outputs
- `get_last_watermark` returns the last successfully persisted value, or `null` if this is the table's first run.
- `advance_watermark` returns a confirmation (success/failure), never silently no-ops.

## Storage
Watermark values are persisted in a dedicated runtime tracking table (e.g., `watermark_registry`), NOT inside `Source_Config` itself — `Source_Config` defines which column to use as the watermark; this registry tracks the actual last-processed value per table. This keeps static config (design-time-ish) separate from constantly-changing runtime state.

| Field | Type | Description |
|---|---|---|
| `table_name` | string | Matches `Source_Config.table_name` |
| `watermark_type` | enum(timestamp, cdc_version) | Which kind of value this is |
| `current_value` | string | The last successfully processed watermark/CDC version |
| `last_updated_at` | timestamp | When this was last advanced |
| `last_load_id` | string | The `_load_id` associated with the last advancement, for audit traceability |

## Implementation Rules
- `advance_watermark` must be called only after `Raw_Framework.md` confirms a successful write — never called speculatively before the write completes.
- `advance_watermark` must be atomic — either the update succeeds fully or not at all, never left in a partially-updated state.
- First-run behavior (`get_last_watermark` returns null) must be handled by the calling component (Incremental/CDC) using a configured default starting point, not by this component inventing one.

## Error Handling
| Scenario | Behavior |
|---|---|
| `watermark_registry` unreachable | Treated as transient, retry per `Error_Handling_Framework.md` |
| Attempting to advance to a value earlier than the current stored value | Reject the advancement, raise `WatermarkRegressionError` — this would indicate a bug or out-of-order execution, never allowed silently |

## Logging
Logs every `advance_watermark` call with `table_name`, old value, new value, and `load_id`. Per `Audit_Framework.md`'s requirement that watermark fields in audit records match actual runtime state exactly.

## Return Values
`get_last_watermark` returns the value or `null`. `advance_watermark` returns success confirmation or raises `WatermarkRegressionError`/transient exceptions.

## Pseudo Logic
```
FUNCTION get_last_watermark(table_name, watermark_type):
    row = SELECT current_value FROM watermark_registry
          WHERE table_name = table_name AND watermark_type = watermark_type
    RETURN row.current_value IF row EXISTS ELSE NULL

FUNCTION advance_watermark(table_name, new_value, load_id):
    current = get_last_watermark(table_name, watermark_type)
    IF current IS NOT NULL AND new_value < current:
        RAISE WatermarkRegressionError(table_name, current, new_value)

    UPSERT watermark_registry
        SET current_value = new_value, last_updated_at = now(), last_load_id = load_id
        WHERE table_name = table_name AND watermark_type = watermark_type

    LOG(table_name, old_value=current, new_value=new_value, load_id=load_id)
```

## Acceptance Criteria
- [ ] Watermark advancement is only ever called after a confirmed successful write — never speculative.
- [ ] Regression (advancing backward) is impossible — always rejected with a clear error.
- [ ] First-run (null watermark) case is handled gracefully by calling components, not this one.
- [ ] Same interface serves both timestamp watermarks and CDC version tracking — no duplicated logic between the two.

## Example (Illustrative Only)
```
table_name: Orders
watermark_type: cdc_version
Before: 000000A1:0000A0E0:0002
After successful write: 000000A2:0000A1F0:0003
load_id: load_20260713_0200_orders
```

## Dependencies
- `Ingestion_Framework.md` (v1.0) — this component implements the watermark/CDC version get-and-advance mechanism referenced there.
- `Config_Framework.md` (v1.0) — `table_name` join key traces back to `Source_Config`.
- `Audit_Framework.md` (v1.0) — watermark values recorded here must match what's captured in audit records exactly.

## Future Extension Points
- Could add a watermark history table (not just current value) if point-in-time watermark auditing becomes a requirement beyond what `Audit_Framework.md` already captures via `last_load_id`.

## AI Generation Notes
Any agent generating Incremental or CDC notebooks must call this component's `get_last_watermark` before querying source, and `advance_watermark` only after `Raw_Framework.md` confirms the write succeeded — never inline custom watermark-tracking logic per table.

## Resolves Forward References From
- `Incremental_Load_Component.md` (v1.0)
- `CDC_Component.md` (v1.0)

Both files' dependency headers should now be treated as fully resolved — no further action needed on those two files.