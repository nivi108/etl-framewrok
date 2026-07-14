# Type Casting Component

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** Config_Framework.md (v1.0), Silver_Framework.md (v1.0)
**Category:** Components

## Purpose
Implements the type casting logic defined in `Silver_Framework.md` stage 3 — casting Raw's landed types to Silver's business types, using `column_type_map` from `Pipeline_Config`, and rejecting rows where a cast fails rather than allowing corrupted or nulled-out values to pass silently.

## Scope
Covers type casting only, runs after `Null_Handling_Component.md` (stage 2) and before `Dedup_Component.md` (stage 4). Does NOT handle null substitution (already resolved by the prior stage) — a null value reaching this stage is either legitimately nullable or already defaulted.

## Inputs
| Input | Type | Description |
|---|---|---|
| `batch` | DataFrame | Output of `Null_Handling_Component`, accepted rows only |
| `column_type_map` | array[object] | From `Pipeline_Config`: `{column_name: string, target_data_type: string}` |

## Outputs
`{accepted: DataFrame, rejected: DataFrame}` — rows split into successful casts and cast failures.

## Configuration
`column_type_map` sourced entirely from `Pipeline_Config` — no hardcoded per-table type logic.

## Implementation Rules
- Columns not listed in `column_type_map` are passed through with their Raw-landed type unchanged — this component only acts on explicitly configured columns.
- A cast failure (e.g., a string that can't convert to the target numeric/date type) results in row rejection, not a silent null substitution — silently nulling a failed cast would hide a real data quality issue.
- Casts must be attempted using safe/try-cast semantics (returning null on failure, which this component then detects) rather than throwing an unhandled exception that could crash the whole batch.

## Error Handling
| Scenario | Behavior |
|---|---|
| Cast produces null where source value was not null (i.e., cast genuinely failed) | Reject row, annotate `_rejection_reason` with column and attempted type |
| `target_data_type` value is itself invalid/unsupported (config error, not data error) | Raise immediately — this needs a config fix, not a per-row skip |

## Logging
Logs per-batch: count of rows rejected for cast failures, broken down by column.

## Return Values
Returns `{accepted: DataFrame, rejected: DataFrame}`, consistent with `Null_Handling_Component.md`'s output shape so both stages can be chained uniformly.

## Pseudo Logic
```
FUNCTION cast_types(batch, column_type_map):
    accepted = []
    rejected = []

    FOR each row in batch:
        row_rejected = false
        FOR each mapping in column_type_map:
            original_value = row[mapping.column_name]
            IF original_value IS NOT NULL:
                cast_value = TRY_CAST(original_value, mapping.target_data_type)
                IF cast_value IS NULL:   # cast genuinely failed
                    row_rejected = true
                    row._rejection_reason = f"cast failure: {mapping.column_name} to {mapping.target_data_type}"
                ELSE:
                    row[mapping.column_name] = cast_value

        IF row_rejected:
            rejected.append(row)
        ELSE:
            accepted.append(row)

    RETURN {accepted: accepted, rejected: rejected}
```

## Acceptance Criteria
- [ ] Every column in `column_type_map` is cast using safe/try-cast semantics, never an unhandled exception.
- [ ] A genuine cast failure always results in rejection, never silent null substitution.
- [ ] Columns not listed in `column_type_map` are left untouched.

## Example (Illustrative Only)
```
column_type_map: [{column_name: "order_date", target_data_type: "date"}]
Row has order_date = "2026-13-45" (invalid date string)
Result: cast fails, row rejected, reason="cast failure: order_date to date"
```

## Dependencies
- `Silver_Framework.md` (v1.0) — implements stage 3 of the Silver pipeline.
- `Config_Framework.md` (v1.0) — `column_type_map` sourced from `Pipeline_Config`, per the structure agreed for column-level metadata.

## Future Extension Points
- Could add configurable cast failure tolerance (e.g., "allow up to 1% cast failure rate before halting the table") if some sources are known to have minor recurring data quality issues that shouldn't block the whole pipeline.

## AI Generation Notes
Any agent generating this component must use try-cast (null-on-failure) semantics and explicit rejection — never a cast operation that could throw and crash the entire batch, and never a cast failure that gets silently coerced to null without rejection.