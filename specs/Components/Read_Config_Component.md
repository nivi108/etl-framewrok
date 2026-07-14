# Read Config Component

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** Config_Framework.md (v1.0)
**Category:** Components

## Purpose
A single, reusable function/module that reads and joins all four config tables (`Source_Config`, `Pipeline_Config`, `Workflow_Config`, `Validation_Config`) for a given table, returning one unified config object. Every other notebook and component that needs configuration reads it through this component — never via ad hoc direct queries against config tables.

## Scope
Covers only the reading and joining of config metadata. Does NOT validate the config's correctness (that's a design-time concern handled during onboarding) and does NOT cache config across runs (each invocation reads fresh, since config can change between runs).

## Inputs
| Input | Type | Description |
|---|---|---|
| `table_name` | string | The table to fetch configuration for |
| `active_only` | boolean, default true | Whether to filter to `active_flag = true` only |

## Outputs
A single structured object (e.g., a Python dict or dataclass) containing all fields from all four config tables for the given `table_name`, joined per the relationships defined in `Config_Framework.md`.

```python
{
    "source_config": {...},       # fields from Source_Config
    "pipeline_config": [...],     # list, since one-to-many across layers
    "workflow_config": {...},     # fields from Workflow_Config
    "validation_config": [...]    # list, since one-to-many
}
```

## Configuration
None — this component takes no static configuration of its own; it purely reads from the four Delta config tables.

## Implementation Rules
- Must always read live from the Delta config tables — no local caching across notebook runs, since a config change (e.g., toggling `active_flag`) must take effect on the very next run without requiring a redeploy.
- If `table_name` has no matching row in `Source_Config`, raise a clear, named exception (`ConfigNotFoundError`) rather than returning an empty/null object silently.
- If `active_only = true` and the table's `active_flag = false`, return a result flagged as `is_active: false` rather than raising an error — calling code decides whether to skip or proceed.

## Error Handling
| Scenario | Behavior |
|---|---|
| `table_name` not found in `Source_Config` | Raise `ConfigNotFoundError` |
| `table_name` found but missing a `Pipeline_Config` row for a required layer | Raise `IncompleteConfigError`, naming the missing layer |
| Config tables themselves unreachable (connectivity issue) | Raise, retry per `Error_Handling_Framework.md` transient-failure rules |

## Logging
Logs a single entry per invocation: `table_name`, `active_only`, whether config was found, and (if not found) the specific error type. Per `Logging_Framework.md`'s Notebook Log type.

## Return Values
Returns the structured config object described in Outputs, or raises a named exception — never returns `None`/`null` silently on failure.

## Dependencies
- `Config_Framework.md` (v1.0) — this component is a direct implementation of the join logic described in that document's Pseudo Logic section (`get_active_pipeline_config`).

## Pseudo Logic
```
FUNCTION read_config(table_name, active_only=true):
    source = SELECT * FROM Source_Config WHERE table_name = table_name
    IF source IS NULL:
        RAISE ConfigNotFoundError(table_name)

    IF active_only AND NOT source.active_flag:
        RETURN {source_config: source, is_active: false}

    pipeline = SELECT * FROM Pipeline_Config WHERE table_name = table_name
    workflow = SELECT * FROM Workflow_Config WHERE table_name = table_name
    validation = SELECT * FROM Validation_Config WHERE table_name = table_name

    IF pipeline IS EMPTY:
        RAISE IncompleteConfigError(table_name, missing="Pipeline_Config")

    RETURN {
        source_config: source,
        pipeline_config: pipeline,
        workflow_config: workflow,
        validation_config: validation,
        is_active: true
    }
```

## Acceptance Criteria
- [ ] Returns a complete, correctly joined config object for any valid, active table.
- [ ] Raises named, specific exceptions rather than generic errors or silent nulls.
- [ ] Never caches stale config across runs.

## Example Metadata (Illustrative Only)
```
Input: table_name="Orders", active_only=true
Output: {source_config: {...load_type: "CDC"...}, pipeline_config: [{target_layer: "Silver",...}, {target_layer: "Gold",...}], ...}
```

## Future Extension Points
- Could add an optional short-lived in-memory cache scoped to a single workflow run (not across runs) if config-read performance becomes a bottleneck with very high table counts.

## AI Generation Notes
Every generated notebook that needs configuration must call this component rather than writing its own query against the config tables — this is what keeps config-reading logic consistent and centrally fixable if the config schema changes.