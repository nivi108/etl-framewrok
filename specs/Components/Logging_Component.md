# Logging Component

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** Config_Framework.md (v1.0), Logging_Framework.md (v1.0)
**Category:** Components

## Purpose
Implements the log-writing functions defined in `Logging_Framework.md` — providing a single, shared interface every notebook and component calls for execution logs, notebook logs, workflow logs, error logs, and performance logs. This is the component that enforces the consistent field names and structured logging the Framework requires.

## Scope
Covers log record writing only. Does NOT define the log schema (that's `Logging_Framework.md`) and does NOT decide *when* to log (each calling notebook is responsible for calling this component at the required logging points defined per layer in `Logging_Framework.md`).

## Inputs (Per Log Type)

### `log_execution_start`
| Input | Type | Description |
|---|---|---|
| `pipeline_id` | string | Overall pipeline run identifier |
| `workflow_id` | string | Workflow identifier |
| `notebook_id` | string | Notebook identifier |
| `table_name` | string | Table being processed |

### `log_execution_end`
| Input | Type | Description |
|---|---|---|
| `pipeline_id` | string | Same as start |
| `notebook_id` | string | Same as start |
| `status` | enum(Success, Failed, Skipped) | Terminal status |
| `counts` | object | `{source, inserted, updated, deleted, rejected}` — whichever apply |

### `log_notebook_step`
| Input | Type | Description |
|---|---|---|
| `pipeline_id` | string | Same as execution |
| `notebook_id` | string | Same as execution |
| `message` | string | Description of the step completed (e.g., "schema validation complete") |

### `log_error`
| Input | Type | Description |
|---|---|---|
| `pipeline_id` | string | Same as execution |
| `notebook_id` | string | Same as execution |
| `error_message` | string | Human-readable error summary |
| `error_stack_trace` | string | Full stack trace for debugging |

### `log_performance`
| Input | Type | Description |
|---|---|---|
| `pipeline_id` | string | Same as execution |
| `notebook_id` | string | Same as execution |
| `start_time` | timestamp | When the measured operation began |
| `end_time` | timestamp | When it completed |

## Outputs
Each function writes a row to the central log table (per the schema in `Logging_Framework.md`) and returns a confirmation. No function returns the log content itself — logs are write-only from the caller's perspective.

## Configuration
No independent configuration. Log table location is a framework-wide constant (catalog/schema/table reference), not per-table or per-notebook.

## Implementation Rules
- Every log write must populate `log_id` (UUID, generated here, not by the caller), `log_type`, and `pipeline_id` at minimum — partial log records missing these fields are rejected by this component before write.
- Sensitive data must never appear in `message` or `error_message` fields — this component does not sanitize (that's the caller's responsibility), but the rule must be documented clearly so AI agents generating notebooks know to sanitize before calling.
- Log writes must be fire-and-forget from the caller's perspective — a log-write failure must never propagate upward as an exception that could halt the pipeline. Internally, this component retries once on failure, then silently drops the log entry if the retry also fails (preferable to halting a pipeline over a lost log line).
- Every `log_execution_start` must be paired with a `log_execution_end` — the component itself does not enforce this pairing, but it must be documented as a contractual obligation on the caller.

## Error Handling
| Scenario | Behavior |
|---|---|
| Log table unreachable | Retry once; if still unreachable, swallow the error and return silently — never halt the pipeline |
| Caller passes null `pipeline_id` or `notebook_id` | Reject the log write, return a warning — do not write a log record that can't be correlated to anything |
| Caller passes sensitive data in `message` | Not caught here — this is a caller responsibility, enforced by AI generation rules and code review |

## Logging
This component does NOT log its own activity (that would be recursive and pointless). Its sole output is the log records it writes for its callers.

## Return Values
Returns `{success: boolean, log_id: string (if written)}`. Never raises exceptions to the caller.

## Pseudo Logic
```
FUNCTION log_execution_start(pipeline_id, workflow_id, notebook_id, table_name):
    IF pipeline_id IS NULL OR notebook_id IS NULL:
        LOG_INTERNAL_WARNING("missing required identifiers, skipping log write")
        RETURN {success: false}

    log_id = NEW_UUID()
    TRY:
        WRITE log_table(
            log_id=log_id, log_type="Execution", status="Started",
            pipeline_id=pipeline_id, workflow_id=workflow_id,
            notebook_id=notebook_id, table_name=table_name,
            start_time=NOW()
        )
        RETURN {success: true, log_id: log_id}
    CATCH:
        RETRY_ONCE()
        RETURN {success: false}

FUNCTION log_execution_end(pipeline_id, notebook_id, status, counts):
    log_id = NEW_UUID()
    TRY:
        WRITE log_table(
            log_id=log_id, log_type="Execution", status=status,
            pipeline_id=pipeline_id, notebook_id=notebook_id,
            end_time=NOW(),
            row_count_source=counts.source, row_count_inserted=counts.inserted,
            row_count_updated=counts.updated, row_count_deleted=counts.deleted,
            row_count_rejected=counts.rejected
        )
        WRITE log_table(
            log_id=NEW_UUID(), log_type="Performance",
            pipeline_id=pipeline_id, notebook_id=notebook_id,
            end_time=NOW(), duration_seconds=COMPUTE_DURATION()
        )
        RETURN {success: true, log_id: log_id}
    CATCH:
        RETRY_ONCE()
        RETURN {success: false}

FUNCTION log_notebook_step(pipeline_id, notebook_id, message):
    TRY:
        WRITE log_table(log_id=NEW_UUID(), log_type="Notebook",
                         pipeline_id=pipeline_id, notebook_id=notebook_id, message=message)
    CATCH:
        RETURN {success: false}   # silently swallowed

FUNCTION log_error(pipeline_id, notebook_id, error_message, error_stack_trace):
    TRY:
        WRITE log_table(log_id=NEW_UUID(), log_type="Error",
                         pipeline_id=pipeline_id, notebook_id=notebook_id,
                         error_message=error_message, error_stack_trace=error_stack_trace)
    CATCH:
        RETURN {success: false}   # best-effort, never blocks
```

## Acceptance Criteria
- [ ] Every log-write function is fire-and-forget — never propagates exceptions to the caller.
- [ ] Every written record includes `log_id`, `log_type`, and `pipeline_id` — incomplete records are rejected before write.
- [ ] No sensitive data passes through this component's fields (caller responsibility, enforced via generation rules).
- [ ] Performance log is automatically written as part of `log_execution_end`, not a separate call the caller must remember.

## Example (Illustrative Only)
```
log_execution_start(pipeline_id="pipe_001", workflow_id="wf_sales", notebook_id="nb_load_orders", table_name="Orders")
→ writes: {log_id: "abc...", log_type: "Execution", status: "Started", start_time: 2026-07-13T02:00:00, ...}

log_execution_end(pipeline_id="pipe_001", notebook_id="nb_load_orders", status="Success", counts={source:15234, inserted:120, updated:45, deleted:0, rejected:2})
→ writes: Execution log + Performance log (duration auto-computed)
```

## Dependencies
- `Logging_Framework.md` (v1.0) — this component implements the write functions for the log schema defined there.
- `Config_Framework.md` (v1.0) — `pipeline_id`, `workflow_id`, `notebook_id` trace to `Workflow_Config`.

## Future Extension Points
- Could add log-level filtering (e.g., suppress `Notebook` step logs in production but keep them in dev) if log volume becomes a concern.
- Could support streaming log records to an external observability tool alongside the Delta log table write.

## AI Generation Notes
Any agent generating notebooks must call `log_execution_start` at the very beginning and `log_execution_end` in a `try/finally` block at the very end — this is the minimum logging contract. Step-level `log_notebook_step` calls at each major processing stage are required per the logging points defined in `Logging_Framework.md`'s per-layer table.