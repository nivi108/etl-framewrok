# Dedup Component

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** Config_Framework.md (v1.0), Silver_Framework.md (v1.0)
**Category:** Components

## Purpose
Implements the deduplication logic defined in `Silver_Framework.md` stage 4 — resolving duplicate rows both within an incoming batch and against already-merged Silver data, using `merge_keys` and a tie-breaking column (watermark or CDC version) to determine which row wins.

## Scope
Covers deduplication decision logic only. Does NOT perform the final merge/write to Silver (that's a separate merge step in `Silver_Framework.md`'s pipeline, stage 8) — this component returns a deduplicated DataFrame ready for that step.

## Inputs
| Input | Type | Description |
|---|---|---|
| `batch` | DataFrame | Incoming batch, post null-handling and type-casting (stages 2-3 already applied) |
| `merge_keys` | array[string] | From `Pipeline_Config`, defines what constitutes a duplicate |
| `tie_break_column` | string | Column used to determine which duplicate wins (typically `watermark_column` or CDC version) |
| `existing_silver_ref` | table reference | Reference to the existing Silver table, for cross-batch dedup check |

## Outputs
A DataFrame with exactly one row per unique `merge_keys` combination — the "winning" row per key, both within the batch and relative to existing Silver data.

## Configuration
No additional configuration beyond the inputs above — all dedup behavior is driven by `merge_keys` and `tie_break_column`, both sourced from config.

## Implementation Rules
- **Intra-batch dedup**: if multiple rows in the incoming batch share the same `merge_keys` value, keep only the row with the highest `tie_break_column` value.
- **Cross-batch dedup**: after intra-batch resolution, compare each surviving row against the corresponding existing Silver row (if any). If the existing Silver row's `tie_break_column` value is already higher or equal, the incoming row is a stale re-delivery and is dropped (not merged).
- Rows with a null `tie_break_column` value are never automatically preferred — if a null is encountered, this is flagged as a data quality issue for that row (routed to DQ rejection, not silently resolved by arbitrary row selection).

## Error Handling
| Scenario | Behavior |
|---|---|
| `tie_break_column` is null for a duplicate group | Flag affected rows for DQ rejection rather than guessing a winner |
| No duplicates present in the batch at all | Valid, common case — passes through unchanged, not an error |

## Logging
Logs `table_name`, count of duplicate groups found, count of rows dropped (both intra-batch and cross-batch), and count of rows flagged for null-tie-break rejection.

## Return Values
Returns the deduplicated DataFrame, plus a count of dropped/flagged rows for audit purposes (consumed by `Audit_Framework.md`'s `dedup_count` field).

## Pseudo Logic
```
FUNCTION deduplicate(batch, merge_keys, tie_break_column, existing_silver_ref):
    # Intra-batch
    windowed = PARTITION_BY(batch, merge_keys), ORDER_BY(tie_break_column DESC)
    intra_deduped = KEEP_FIRST_PER_PARTITION(windowed)
    dropped_intra_count = batch.count() - intra_deduped.count()

    # Flag nulls in tie_break_column among duplicate groups
    null_tiebreak_rows = FILTER(intra_deduped, tie_break_column IS NULL AND had_duplicates)
    intra_deduped = intra_deduped - null_tiebreak_rows

    # Cross-batch against existing Silver
    joined = LEFT_JOIN(intra_deduped, existing_silver_ref, ON merge_keys)
    final = FILTER(joined, existing_silver_ref.tie_break_column IS NULL
                            OR intra_deduped.tie_break_column > existing_silver_ref.tie_break_column)
    dropped_cross_count = intra_deduped.count() - final.count()

    LOG(dropped_intra_count, dropped_cross_count, null_tiebreak_count=null_tiebreak_rows.count())
    RETURN {data: final, dedup_count: dropped_intra_count + dropped_cross_count, rejected: null_tiebreak_rows}
```

## Acceptance Criteria
- [ ] Exactly one row survives per unique `merge_keys` value after processing.
- [ ] Stale re-deliveries (already superseded by existing Silver data) are correctly dropped, not re-merged.
- [ ] Null tie-break values among duplicates are never silently resolved by arbitrary selection — always flagged for DQ review.

## Example (Illustrative Only)
```
Incoming batch has 3 rows for order_id=500, watermark values: 10:00, 10:05, 10:03
Result: row with watermark 10:05 wins (intra-batch)
Existing Silver already has order_id=500 at watermark 10:07
Result: incoming winner (10:05) is stale relative to existing (10:07) -> dropped (cross-batch)
```

## Dependencies
- `Silver_Framework.md` (v1.0) — this component implements stage 4 of the Silver pipeline.
- `Config_Framework.md` (v1.0) — `merge_keys` sourced from `Pipeline_Config`.

## Future Extension Points
- Could support composite tie-break logic (e.g., watermark first, then a secondary sequence column) if a single column proves insufficient for some tables.

## AI Generation Notes
Any agent generating deduplication logic must implement both intra-batch and cross-batch checks — implementing only intra-batch dedup (a common oversight) would allow stale re-deliveries to overwrite newer Silver data.