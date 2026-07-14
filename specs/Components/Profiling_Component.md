# Profiling Component

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** Config_Framework.md (v1.0), Silver_Framework.md (v1.0), Logging_Framework.md (v1.0)
**Category:** Components

## Purpose
Implements the data profiling behavior defined in `Silver_Framework.md` — computing row counts, null percentages, distinct value counts, and min/max ranges as an informational side process. Profiling results feed into monitoring but never gate execution, per the "profiling is non-blocking" rule established there.

## Scope
Covers profile computation and result storage only. Does NOT decide what constitutes an "unhealthy" profile (that's a monitoring/alerting concern, potentially handled by `Notification_Component.md` in the future) and does NOT block pipeline execution regardless of what profiling reveals.

## Inputs
| Input | Type | Description |
|---|---|---|
| `batch` | DataFrame | The batch to profile (typically post-cleansing, pre-merge in Silver) |
| `profiling_config` | object, optional | Which columns to profile and which metrics to compute; defaults to profiling all columns with standard metrics if omitted |

## Outputs
A profile record written to a `data_profile_log` table, containing:

| Field | Type | Description |
|---|---|---|
| `profile_id` | string | Unique profile record identifier |
| `pipeline_id` | string | Links to the pipeline run, per `Logging_Framework.md` |
| `table_name` | string | Table being profiled |
| `layer` | enum(Raw, Silver, Gold) | Layer this profile applies to |
| `profiled_at` | timestamp | When this profile was computed |
| `row_count` | integer | Total row count of the profiled batch |
| `column_metrics` | array[object] | Per-column: `{column_name, null_count, null_percentage, distinct_count, min_value, max_value}` |

## Configuration
`profiling_config` is optional; when omitted, defaults are used. When provided, it can restrict profiling to specific columns (useful for very wide tables where full profiling is expensive) or specific metric types (e.g., skip `distinct_count` on columns known to be always-unique like primary keys, since counting distinct values there wastes compute).

## Implementation Rules
- Profiling must never modify the batch — read-only computation, no schema changes, no row filtering.
- Profiling failures (e.g., a metric computation error on one column) must never halt the batch or the overall pipeline — log the specific metric failure, skip that metric, continue.
- Profile computation should be reasonably efficient — avoid full scans where sampling would suffice for large tables (e.g., sample-based distinct count if the batch exceeds a configured size threshold).

## Error Handling
| Scenario | Behavior |
|---|---|
| A single metric fails to compute on one column | Log warning, skip that metric, continue with other metrics/columns |
| The entire profiling operation fails (e.g., `data_profile_log` unreachable) | Log warning, do NOT halt the pipeline — pipeline execution continues without a profile record for this run |

## Logging
Logs profile computation start/end, count of columns profiled, count of metrics skipped due to errors. Actual profile results go into the `data_profile_log` table, not the standard log tables — kept separate to avoid bloating operational logs.

## Return Values
Returns the written profile record (or null if profiling failed entirely — never raises an exception that could propagate to the caller and disrupt pipeline flow).

## Pseudo Logic
```
FUNCTION profile_batch(batch, table_config, layer, pipeline_id, profiling_config=None):
    IF profiling_config IS NULL:
        columns_to_profile = batch.columns   # default: all
        metrics = [null_count, null_percentage, distinct_count, min_value, max_value]
    ELSE:
        columns_to_profile = profiling_config.columns
        metrics = profiling_config.metrics

    profile_record = {
        profile_id: NEW_UUID(),
        pipeline_id: pipeline_id,
        table_name: table_config.table_name,
        layer: layer,
        profiled_at: NOW(),
        row_count: batch.count(),
        column_metrics: []
    }

    FOR each column in columns_to_profile:
        col_metrics = {column_name: column}
        FOR each metric in metrics:
            TRY:
                col_metrics[metric] = COMPUTE(metric, batch, column)
            CATCH:
                LOG_WARNING(f"metric {metric} failed on column {column}, skipping")
        profile_record.column_metrics.append(col_metrics)

    TRY:
        WRITE data_profile_log(profile_record)
    CATCH:
        LOG_WARNING("data_profile_log write failed, continuing without profile")
        RETURN NULL

    RETURN profile_record
```

## Acceptance Criteria
- [ ] Profiling never halts the pipeline, regardless of what it discovers or what fails within it.
- [ ] Profile records are written to a separate `data_profile_log` table, distinct from operational log/audit tables.
- [ ] Batch content is never modified by profiling — strictly read-only.

## Example (Illustrative Only)
```
table_name: Orders
layer: Silver
row_count: 15186
column_metrics:
  - {column_name: order_id, null_count: 0, null_percentage: 0.0, distinct_count: 15186, min: 1, max: 987654}
  - {column_name: discount_code, null_count: 12203, null_percentage: 80.4, distinct_count: 15, min: null, max: null}
```

## Dependencies
- `Silver_Framework.md` (v1.0) — this component implements the profiling side process defined there.
- `Logging_Framework.md` (v1.0) — shares `pipeline_id` for cross-reference, but writes to `data_profile_log`, not standard log tables.
- `Config_Framework.md` (v1.0) — `table_name` traces to `Source_Config`; `profiling_config` may be added as a nested field on `Pipeline_Config` if it becomes commonly overridden per table (currently defaults suffice for most cases).

## Future Extension Points
- Could add trend-based alerting (e.g., "null percentage on column X jumped from 5% to 40% between runs — flag as anomaly"), building on the historical profile records this component creates.
- Could support pluggable custom metrics (percentile distributions, string length distributions) if standard metrics prove insufficient for some tables.

## AI Generation Notes
Any agent generating Silver notebooks must call this component as a non-blocking side step — profiling calls must be wrapped so that any failure is logged and swallowed, never propagated to halt the main pipeline flow.