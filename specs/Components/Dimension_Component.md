# Dimension Component

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** Config_Framework.md (v1.0), Gold_Framework.md (v1.0), Surrogate_Key_Component.md (v1.0), Merge_Component.md (v1.0), SCD_Type2_Framework.md (v1.0)
**Category:** Components

## Purpose
Orchestrates the complete dimension-building process defined in `Gold_Framework.md` — reading from Silver, resolving surrogate keys, applying SCD2 if configured, and merging into the Gold dimension table. This component ties together `Surrogate_Key_Component.md`, `Merge_Component.md`, and (when applicable) SCD2 logic into one cohesive dimension-build routine.

## Scope
Covers dimension-specific orchestration only. Delegates surrogate key resolution to `Surrogate_Key_Component.md` and the actual write to `Merge_Component.md` — this component's value is correctly sequencing those steps for dimensions specifically (as opposed to `Fact_Component.md`, its fact-building counterpart).

## Inputs
| Input | Type | Description |
|---|---|---|
| `table_config` | object | Output of `Read_Config_Component`, for the dimension table being built |
| `silver_data` | DataFrame | Cleansed data from the corresponding Silver table |

## Outputs
Merge execution result (from `Merge_Component.md`), representing rows inserted/updated in the Gold dimension table.

## Configuration
Reads `business_keys`, `merge_keys`, `scd_type` from `table_config.pipeline_config` (the Gold-layer row).

## Implementation Rules
- Every incoming row must have its surrogate key resolved (via `Surrogate_Key_Component`) before any merge attempt — a dimension row is never written without a surrogate key.
- If `scd_type = 2`, the merge mode passed to `Merge_Component` must be `scd2`; if `scd_type = 1` or `None`, use `simple_upsert`.
- Must always ensure an "Unknown member" row exists in the dimension table (surrogate key reserved, e.g., `-1`, with placeholder attribute values) — created once during initial dimension setup, not regenerated on every run. This satisfies `Gold_Framework.md`'s requirement that facts never have null dimension foreign keys.

## Error Handling
Delegates to `Surrogate_Key_Component.md` and `Merge_Component.md` for their respective error conditions. Additionally:
| Scenario | Behavior |
|---|---|
| "Unknown member" row missing from the dimension table at build time | Create it as part of this run (idempotent — checks existence first) rather than failing |

## Logging
Logs `dimension_table`, count of new surrogate keys generated, count of SCD2 versions created (if applicable), merge result summary.

## Return Values
Returns the merge result object from `Merge_Component.md`.

## Pseudo Logic
```
FUNCTION build_dimension(table_config, silver_data):
    ENSURE_UNKNOWN_MEMBER_EXISTS(table_config.dimension_table)

    keyed_data = []
    FOR each row in silver_data:
        surrogate_key = Surrogate_Key_Component.resolve_surrogate_key(
            row[table_config.business_keys], table_config.dimension_table
        )
        row.surrogate_key = surrogate_key
        keyed_data.append(row)

    merge_mode = "scd2" IF table_config.scd_type == 2 ELSE "simple_upsert"

    result = Merge_Component.execute_merge(
        source_data=keyed_data,
        target_table=table_config.dimension_table,
        merge_keys=table_config.merge_keys,
        merge_mode=merge_mode
    )

    LOG(table_config.dimension_table, result)
    RETURN result
```

## Acceptance Criteria
- [ ] No dimension row is ever merged without a resolved surrogate key.
- [ ] "Unknown member" row always exists before any fact table attempts to reference this dimension.
- [ ] Correct merge mode (`scd2` vs `simple_upsert`) is selected based purely on `table_config.scd_type`, never hardcoded per table.

## Example (Illustrative Only)
```
table_config.dimension_table: dim_customer
table_config.scd_type: 2
Result: 3 new surrogate keys generated, 3 SCD2 version rows inserted, "Unknown member" row confirmed present
```

## Dependencies
- `Gold_Framework.md` (v1.0) — this component implements the dimension-building process defined there.
- `Surrogate_Key_Component.md` (v1.0), `Merge_Component.md` (v1.0) — delegated sub-steps.
- `SCD_Type2_Framework.md` (v1.0) — governs merge mode selection when `scd_type = 2`.
- `Config_Framework.md` (v1.0) — supplies `business_keys`, `merge_keys`, `scd_type`.

## Future Extension Points
- Could support snowflake-schema dimension hierarchies (dimension referencing another dimension) as a future variant, per the exception noted in `Gold_Framework.md`.

## AI Generation Notes
Any agent generating a dimension-building notebook must call this component rather than reimplementing surrogate key resolution and merge logic inline — this is the canonical dimension-build routine for the entire framework.