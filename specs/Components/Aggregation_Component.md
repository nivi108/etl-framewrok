# Aggregation Component

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** Config_Framework.md (v1.0), Gold_Framework.md (v1.0), Merge_Component.md (v1.0), Fact_Component.md (v1.0)
**Category:** Components

## Purpose
Implements the aggregation logic defined in `Gold_Framework.md` — building pre-computed summary tables (e.g., daily sales totals, monthly customer revenue) derived from Gold facts. Aggregates exist for query performance, not as a primary source of truth — per the Framework's rule that aggregates must always be reproducible from Facts + Dimensions.

## Scope
Covers aggregate computation and write only. Does NOT define which aggregates to build (that's driven per-table via config), and does NOT decide when to refresh (that's `Workflow_Orchestration_Framework.md`'s scheduling responsibility).

## Inputs
| Input | Type | Description |
|---|---|---|
| `table_config` | object | Output of `Read_Config_Component`, for the aggregate table being built |
| `source_fact_data` | DataFrame | The fact table (or window of it) to aggregate from |
| `aggregate_definition` | object | Group-by columns, measures to aggregate, aggregation functions (sum/count/avg/etc.) |

## Outputs
Merge/write execution result for the aggregate table.

## Configuration
`aggregate_definition` is sourced from `Pipeline_Config` (added alongside `measure_definitions` and other Gold-layer fields, per the same nested-object pattern established in `Config_Framework.md`). Structure:
```yaml
group_by: [column_name, ...]
aggregations: [{measure_name: string, source_column: string, function: enum(sum, count, avg, min, max, count_distinct)}]
refresh_mode: enum(incremental, full_recompute)
```

## Implementation Rules
- Prefer `incremental` refresh mode: recompute only affected group-by partitions (e.g., only "yesterday's" daily totals if only yesterday's facts changed) rather than recomputing the entire aggregate every time.
- `full_recompute` mode is permitted for small aggregates or when incremental logic would be more error-prone than a full rebuild — chosen deliberately per table, not defaulted.
- Aggregates must remain **derivable** — i.e., running a full recompute against the current Fact + Dimension state must always be able to reproduce the aggregate exactly. If a manual "correction" to an aggregate diverges from what facts would produce, that's a governance violation, not a legitimate update.

## Error Handling
| Scenario | Behavior |
|---|---|
| Source fact table doesn't exist or is empty (first-run edge case) | Log and produce an empty aggregate table rather than raising — an empty aggregate is a valid state on first run |
| An aggregation function is invalid/unsupported | Raise immediately — config error, not per-row data issue |

## Logging
Logs `aggregate_table`, `refresh_mode`, count of group-by partitions refreshed, row count written.

## Return Values
Returns the merge/write result from `Merge_Component.md`.

## Pseudo Logic
```
FUNCTION build_aggregate(table_config, source_fact_data, aggregate_definition):
    IF aggregate_definition.refresh_mode == "incremental":
        affected_partitions = IDENTIFY_CHANGED_PARTITIONS(source_fact_data, aggregate_definition.group_by)
        source = FILTER(source_fact_data, partition IN affected_partitions)
    ELSE:
        source = source_fact_data   # full recompute

    aggregated = GROUP_BY(source, aggregate_definition.group_by)
                 .APPLY(aggregate_definition.aggregations)   # sum(quantity), avg(unit_price), etc.

    result = Merge_Component.execute_merge(
        source_data=aggregated,
        target_table=table_config.aggregate_table,
        merge_keys=aggregate_definition.group_by,   # group-by columns are the natural key
        merge_mode="simple_upsert"
    )

    LOG(table_config.aggregate_table, aggregate_definition.refresh_mode, result)
    RETURN result
```

## Acceptance Criteria
- [ ] Aggregates are always reproducible from Facts + Dimensions — never a divergent source of truth.
- [ ] Incremental refresh only touches partitions actually affected by new/changed facts, not the whole table.
- [ ] Group-by columns serve as the natural merge key — no separate surrogate keys needed for aggregates.

## Example (Illustrative Only)
```
aggregate_table: agg_daily_sales
group_by: [order_date, product_key]
aggregations:
  - {measure_name: total_quantity, source_column: quantity, function: sum}
  - {measure_name: total_revenue, source_column: line_total, function: sum}
refresh_mode: incremental
```

## Dependencies
- `Gold_Framework.md` (v1.0) — this component implements the aggregation portion of the Gold layer.
- `Merge_Component.md` (v1.0) — write mechanism.
- `Fact_Component.md` (v1.0) — aggregates are always built from Fact outputs, never Silver directly.
- `Config_Framework.md` (v1.0) — `aggregate_definition` sourced from `Pipeline_Config`.

## Future Extension Points
- Could support rolling/windowed aggregations (e.g., trailing-7-day averages) beyond the standard group-by-and-summarize pattern.
- Could integrate with a materialized view mechanism if Databricks native materialized views become preferable to explicitly built aggregate tables.

## AI Generation Notes
Any agent generating aggregate notebooks must default to incremental refresh mode where feasible and must never allow an aggregate to become a divergent source of truth — a full recompute against current Facts must always reproduce the same aggregate values, and this should be enforceable via a periodic reconciliation check.