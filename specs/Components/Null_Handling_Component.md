# Null Handling Component

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** Config_Framework.md (v1.0), Silver_Framework.md (v1.0)
**Category:** Components

## Purpose
Implements the null handling decision table defined in `Silver_Framework.md` stage 2 — applying defaults where configured, rejecting rows where a non-nullable field is null with no default, and passing through nullable fields unchanged.

## Scope
Covers null evaluation and default substitution only. Does NOT perform type casting (that's `Type_Casting_Component.md`, stage 3, which runs after this) and does NOT decide table-level nullability rules (those come from `null_rules` config, defined per column).

## Inputs
| Input | Type | Description |
|---|---|---|
| `batch` | DataFrame | Incoming batch, post schema validation |
| `null_rules` | object | Per-column rules: `{column_name: {nullable: boolean, default_value: any, optional}}` |

## Outputs
`{accepted: DataFrame, rejected: DataFrame}` — rows split into those that passed null handling and those rejected for a non-nullable field with no default.

## Configuration
`null_rules` is sourced from `Pipeline_Config` (or a dedicated column-level metadata extension of it) — per-column nullability and default values are never hardcoded in this component.

## Implementation Rules
Per the decision table in `Silver_Framework.md`:

| Column Nullability | Value Present? | Action |
|---|---|---|
| Nullable | Null or Present | Accept as-is |
| Non-nullable | Present | Accept as-is |
| Non-nullable | Null, default configured | Apply default, log substitution |
| Non-nullable | Null, no default configured | Reject row |

- Default substitution must be logged per occurrence (column, row identifier, default applied) — silent substitution undermines the auditability principle established in `Logging_Framework.md`.
- A column not present in `null_rules` at all is treated as nullable by default (fail-open on missing config, rather than fail-closed rejecting everything) — but this should trigger a warning log, since an undocumented column's nullability is a config gap worth surfacing.

## Error Handling
| Scenario | Behavior |
|---|---|
| `null_rules` missing entirely for the table | Treat all columns as nullable, log a warning (config incompleteness, not a hard failure) |
| Default value type doesn't match column type | Raise immediately — this is a config error, not a data error |

## Logging
Logs per-batch: count of rows rejected for null violations (by column), count of default substitutions applied (by column).

## Return Values
Returns `{accepted: DataFrame, rejected: DataFrame}`. Rejected rows carry a `_rejection_reason` annotation identifying which column(s) caused rejection.

## Pseudo Logic
```
FUNCTION handle_nulls(batch, null_rules):
    accepted = []
    rejected = []

    FOR each row in batch:
        row_rejected = false
        FOR each column, rule in null_rules:
            IF row[column] IS NULL AND NOT rule.nullable:
                IF rule.default_value IS NOT NULL:
                    row[column] = rule.default_value
                    LOG("default_applied", column, row_id=row.id)
                ELSE:
                    row_rejected = true
                    row._rejection_reason = f"non-nullable column {column} is null"

        IF row_rejected:
            rejected.append(row)
        ELSE:
            accepted.append(row)

    RETURN {accepted: accepted, rejected: rejected}
```

## Acceptance Criteria
- [ ] Every non-nullable column with a null value and no default results in row rejection, never silent acceptance.
- [ ] Every default substitution is logged individually, not just as an aggregate count.
- [ ] Missing `null_rules` config fails open (nullable) with a warning, not a hard error — avoids blocking onboarding over an incomplete config while still surfacing the gap.

## Example (Illustrative Only)
```
Column: discount_code, nullable=false, default_value=""
Row has discount_code=NULL -> default applied, logged, row accepted

Column: order_id, nullable=false, no default
Row has order_id=NULL -> row rejected, reason="non-nullable column order_id is null"
```

## Dependencies
- `Silver_Framework.md` (v1.0) — implements stage 2 of the Silver pipeline exactly as defined there.
- `Config_Framework.md` (v1.0) — `null_rules` sourced from `Pipeline_Config`.

## Future Extension Points
- Could add computed defaults (e.g., "default to yesterday's value for this business key") beyond simple static default values, if business rules require it.

## AI Generation Notes
Any agent generating this component must log every default substitution individually and must never accept a non-nullable-null combination without either a configured default or an explicit rejection — there is no third path.