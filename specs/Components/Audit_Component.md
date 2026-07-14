# Audit Component

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** Config_Framework.md (v1.0), Audit_Framework.md (v1.0), Logging_Framework.md (v1.0)
**Category:** Components

## Purpose
Implements the audit record write and reconciliation check defined in `Audit_Framework.md` — called by every layer's processing at completion, capturing source/target counts, checksums, watermark state, and reconciliation status into the audit table.

## Scope
Covers audit record writing and reconciliation logic only. Does NOT capture the counts themselves (those are supplied by the calling notebook, which already tracks them for its own purposes) and does NOT trigger alerts on mismatch (that's `Notification_Component.md` — this component just flags the record's status).

## Inputs
| Input | Type | Description |
|---|---|---|
| `table_name` | string | Table being audited |
| `layer` | enum(Raw, Silver, Gold) | Layer this audit applies to |
| `pipeline_id` | string | Links to the pipeline run |
| `counts` | object | `{source_count, target_count, rejected_count, dedup_count (Silver only), aggregation_change (Gold only)}` |
| `checksums` | object, optional | `{source_checksum, target_checksum}` when checksums are computed |
| `watermark_state` | object, optional | `{watermark_value, cdc_version}` — whichever applies to the load type |

## Outputs
An audit record written to the audit table, matching the schema defined in `Audit_Framework.md`. Returns the reconciliation status so the caller knows whether to trigger downstream alerting.

## Configuration
No independent configuration — driven entirely by the counts and metadata the caller supplies.

## Implementation Rules
- Every layer's processing must produce exactly one audit record per run, even on partial failure (per `Audit_Framework.md`) — this means this component must be safely callable from within a `finally`/cleanup block, not only on the success path.
- Reconciliation formula depends on layer:
  - Raw: `source_count == target_count + rejected_count`
  - Silver: `source_count == target_count + rejected_count + dedup_count`
  - Gold: `source_count == target_count + rejected_count + aggregation_change` (where `aggregation_change` explicitly models legitimate grain-changing operations)
- A `Mismatch` result must never be silently corrected — it's a signal for investigation, per `Audit_Framework.md`.

## Error Handling
| Scenario | Behavior |
|---|---|
| Audit table unreachable | Retry per `Error_Handling_Framework.md` transient rules; if retries exhaust, log the failure — but do NOT block the pipeline solely on audit-write failure (a completed pipeline without an audit record is preferable to a rolled-back pipeline over an audit-write hiccup) |
| Counts supplied by caller are inconsistent (e.g., `target_count > source_count` with no aggregation) | Still write the record, but flag `reconciliation_status = Mismatch` and log the anomaly — never refuse to write |

## Logging
Logs `table_name`, `layer`, reconciliation status, any exceptional conditions. Per `Logging_Framework.md`'s Notebook Log type.

## Return Values
Returns `{audit_id, reconciliation_status: enum(Match, Mismatch, NotApplicable)}`.

## Pseudo Logic
```
FUNCTION write_audit(table_name, layer, pipeline_id, counts, checksums=None, watermark_state=None):
    expected_target = counts.source_count - counts.rejected_count
    IF layer == "Silver":
        expected_target -= counts.dedup_count
    IF layer == "Gold" AND counts.aggregation_change IS SET:
        expected_target -= counts.aggregation_change   # legitimate row-count change

    status = "Match" IF counts.target_count == expected_target ELSE "Mismatch"

    IF checksums IS SET AND checksums.source_checksum != checksums.target_checksum:
        status = "Mismatch"

    audit_id = NEW_UUID()
    WRITE audit_record(
        audit_id=audit_id,
        table_name=table_name,
        layer=layer,
        pipeline_id=pipeline_id,
        source_count=counts.source_count,
        target_count=counts.target_count,
        rejected_count=counts.rejected_count,
        dedup_count=counts.dedup_count,
        watermark_value=watermark_state.watermark_value,
        cdc_version=watermark_state.cdc_version,
        checksum_source=checksums.source_checksum,
        checksum_target=checksums.target_checksum,
        reconciliation_status=status
    )

    LOG(table_name, layer, status)
    RETURN {audit_id, reconciliation_status: status}
```

## Acceptance Criteria
- [ ] Exactly one audit record is written per layer-per-run, even on partial failure.
- [ ] Reconciliation formula correctly accounts for layer-specific legitimate row-count changes.
- [ ] A `Mismatch` status is flagged but never silently corrected.
- [ ] Audit-write failure never rolls back or blocks a completed pipeline.

## Example (Illustrative Only)
```
table_name: Orders
layer: Silver
counts: {source_count: 15234, target_count: 15186, rejected_count: 3, dedup_count: 45}
Result: reconciliation_status = Match  (15234 - 3 - 45 = 15186)
```

## Dependencies
- `Audit_Framework.md` (v1.0) — this component implements the audit record write defined there.
- `Logging_Framework.md` (v1.0) — for cross-reference of `pipeline_id` and log correlation.
- `Config_Framework.md` (v1.0) — `table_name` traces to `Source_Config`.

## Future Extension Points
- Could add a "reconciliation retry" mechanism: on `Mismatch`, automatically re-count from source and target and compare a second time to rule out transient counting inconsistencies before flagging.
- Could integrate with `Notification_Component.md` directly (currently just returns status; caller decides whether to alert).

## AI Generation Notes
Any agent generating layer-processing notebooks must call this component in a way that guarantees it runs even on failure (e.g., wrap in try/finally). An audit record with `status=Mismatch` and low counts is more useful than no audit record at all.