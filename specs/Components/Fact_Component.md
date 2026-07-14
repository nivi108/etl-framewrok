# Fact Component

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** Config_Framework.md (v1.0), Gold_Framework.md (v1.0), Surrogate_Key_Component.md (v1.0), Merge_Component.md (v1.0), Dimension_Component.md (v1.0)
**Category:** Components

## Purpose
Orchestrates the complete fact-building process defined in `Gold_Framework.md` — resolving dimension foreign keys, applying measure calculations, and writing to the Gold fact table. This is `Dimension_Component.md`'s counterpart for fact tables.

## Scope
Covers fact-specific orchestration only. Delegates dimension key lookup to `Surrogate_Key_Component.md` and the write to `Merge_Component.md`. Assumes all referenced dimensions have already been built by `Dimension_Component.md` in this run — per the dependency ordering enforced in `Workflow_Orchestration_Framework.md`.

## Inputs
| Input | Type | Description |
|---|---|---|
| `table_config` | object | Output of `Read_Config_Component`, for the fact table being built |
| `silver_data` | DataFrame | Cleansed transactional data from the corresponding Silver table |
| `dimension_refs` | array[object] | Which dimensions this fact references, and which business key column maps to each |

## Outputs
Merge/write execution result (from `Merge_Component.md`), representing rows inserted (or merged) into the Gold fact table.

## Configuration
Reads `dimension_refs`, `measure_definitions`, `fact_write_mode` from `table_config.pipeline_config` (the Gold-layer row).

## Implementation Rules
- For every row, every dimension reference listed in `dimension_refs` must be resolved to a surrogate key via `Surrogate_Key_Component.resolve_surrogate_key` — using lookup-only behavior (a fact should never trigger new dimension row generation; dimensions must already exist by the time facts run, per the orchestration dependency order).
- If a business key referenced by a fact row doesn't exist in the dimension (a true late-arriving fact edge case, despite normal ordering), resolve to the "Unknown member" surrogate key rather than failing the row — this was reserved specifically for this purpose in `Dimension_Component.md`.
- Measures are computed per `measure_definitions` expressions (e.g., `line_total = quantity * unit_price`) — these expressions must be simple, declarative, and evaluable generically (consistent with the declarative-expression principle established in `Data_Quality_Framework.md`).
- Grain must be explicitly respected: the row-level granularity of the output must match what's documented for this fact table (e.g., "one row per order line item") — this component does not enforce grain automatically; it trusts the Silver source data already matches the intended grain, and any grain violation is a config/design issue caught during review, not a runtime check here.

## Error Handling
| Scenario | Behavior |
|---|---|
| Dimension business key not found (late-arriving fact) | Resolve to "Unknown member" key, log occurrence, continue (does not reject the row) |
| Measure expression fails to evaluate (e.g., missing column referenced in the expression) | Raise immediately — this is a config error in `measure_definitions`, not a per-row data issue |

## Logging
Logs `fact_table`, count of rows resolved to "Unknown member" per dimension (worth monitoring — a high rate suggests an ordering or data quality problem), measure computation summary, merge/write result.

## Return Values
Returns the merge/write result object from `Merge_Component.md`.

## Pseudo Logic
```
FUNCTION build_fact(table_config, silver_data, dimension_refs):
    resolved_data = []
    FOR each row in silver_data:
        FOR each dim_ref in dimension_refs:
            business_key_value = row[dim_ref.source_column]
            surrogate_key = Surrogate_Key_Component.lookup_only(business_key_value, dim_ref.dimension_table)
            IF surrogate_key IS NULL:
                surrogate_key = UNKNOWN_MEMBER_KEY(dim_ref.dimension_table)
                LOG_WARNING("unresolved dimension key, using Unknown member", dim_ref.dimension_table)
            row[dim_ref.foreign_key_column] = surrogate_key

        FOR each measure in table_config.measure_definitions:
            row[measure.measure_name] = EVALUATE(measure.expression, row)

        resolved_data.append(row)

    result = Merge_Component.execute_merge(
        source_data=resolved_data,
        target_table=table_config.fact_table,
        merge_keys=table_config.merge_keys,
        merge_mode="simple_upsert" IF table_config.fact_write_mode == "merge" ELSE "append"
    )

    LOG(table_config.fact_table, result)
    RETURN result
```

## Acceptance Criteria
- [ ] No fact row is ever written with a null dimension foreign key — always resolves to a real surrogate key or "Unknown member."
- [ ] Fact building never triggers new dimension row generation — strictly lookup-only against already-built dimensions.
- [ ] Measure calculations use declarative expressions, evaluated generically rather than hardcoded per table.

## Example (Illustrative Only)
```
table_config.fact_table: fact_sales
dimension_refs: [{dimension_table: dim_customer, source_column: customer_id, foreign_key_column: customer_key}, ...]
measure_definitions: [{measure_name: line_total, expression: "quantity * unit_price"}]
Result: all rows resolved with customer_key populated (real or Unknown member), line_total computed, written to fact_sales
```

## Dependencies
- `Gold_Framework.md` (v1.0) — this component implements the fact-building process defined there.
- `Surrogate_Key_Component.md` (v1.0) — dimension key lookup (lookup-only mode for facts).
- `Merge_Component.md` (v1.0) — the write/merge mechanism.
- `Dimension_Component.md` (v1.0) — facts assume dimensions built by this component already exist.
- `Config_Framework.md` (v1.0) — supplies `dimension_refs`, `measure_definitions`, `fact_write_mode`.

## Future Extension Points
- Could support degenerate dimensions (attributes stored directly on the fact without a separate dimension table) if a future fact table needs that pattern.

## AI Generation Notes
Any agent generating a fact-building notebook must use lookup-only dimension key resolution (never generate new dimension rows from within fact logic) and must always fall back to the "Unknown member" key rather than nulling or rejecting a row solely due to an unresolved dimension reference.