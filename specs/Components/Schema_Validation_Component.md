# Schema Validation Component

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** Config_Framework.md (v1.0), Schema_Management_Framework.md (v1.0)
**Category:** Components

## Purpose
Implements the schema comparison and drift detection logic defined in `Schema_Management_Framework.md` — comparing an incoming batch's schema against the last known schema, classifying the difference, and returning a pass/accommodate/halt decision.

## Scope
Covers schema comparison logic only. Does NOT decide what happens after a halt is raised (that's `Error_Handling_Framework.md`) and does NOT store the schema registry itself beyond reading/writing it (no business logic about retention, versioning policy beyond what's already defined in the Framework).

## Inputs
| Input | Type | Description |
|---|---|---|
| `table_name` | string | Table being validated |
| `incoming_schema` | schema object | Schema of the current batch (column names, types, nullability) |

## Outputs
A classification result: `{status: enum(Pass, AccommodatedPass, Halt), diff: object, new_schema_version: integer (if changed)}`

## Configuration
None beyond reading the current schema registry entry for the given `table_name`.

## Implementation Rules
- Comparison must check: added columns, removed columns, type changes, nullability changes — each classified independently per the decision table in `Schema_Management_Framework.md`.
- If ANY single change in the diff is classified as breaking (removal, narrowing, rename-appearing-as-drop+add), the overall result is `Halt`, even if other changes in the same diff are safely accommodate-able. Partial accommodation of a mixed diff is not permitted — an all-or-nothing decision per batch.
- On `AccommodatedPass`, this component is responsible for writing the updated schema to `schema_registry` with an incremented `schema_version`. On `Halt`, the registry is NOT updated (so the next run compares against the same last-known-good schema, not the rejected one).

## Error Handling
| Scenario | Behavior |
|---|---|
| No prior schema registry entry exists (first run for this table) | Treat as `Pass`, write initial schema to registry as `schema_version = 1` |
| Registry read/write failure | Transient, retry per `Error_Handling_Framework.md` |

## Logging
Logs `table_name`, diff summary, resulting status, and (if changed) old/new `schema_version`.

## Return Values
Returns the classification result object described in Outputs. Never raises an exception for a detected breaking change — that's a valid, expected outcome (`Halt` status), not an error condition in the component itself. Only infrastructure failures (registry unreachable) raise exceptions.

## Pseudo Logic
```
FUNCTION validate_schema(table_name, incoming_schema):
    last_known = READ schema_registry WHERE table_name = table_name

    IF last_known IS NULL:
        WRITE schema_registry(table_name, incoming_schema, schema_version=1)
        RETURN {status: "Pass", diff: null, new_schema_version: 1}

    diff = COMPUTE_DIFF(incoming_schema, last_known.column_definitions)

    IF diff.is_empty:
        RETURN {status: "Pass", diff: null}

    IF diff.has_any_breaking_change:   # removal, narrowing, rename
        RETURN {status: "Halt", diff: diff}

    # only additive/safe-widening changes present
    new_version = last_known.schema_version + 1
    WRITE schema_registry(table_name, incoming_schema, schema_version=new_version)
    RETURN {status: "AccommodatedPass", diff: diff, new_schema_version: new_version}
```

## Acceptance Criteria
- [ ] A mixed diff (some safe, some breaking changes) always results in `Halt`, never partial accommodation.
- [ ] Registry is only updated on `Pass` (first run) or `AccommodatedPass` — never on `Halt`.
- [ ] First-run behavior correctly initializes the registry rather than treating it as a diff against nothing.

## Example (Illustrative Only)
```
table_name: Orders
Diff: + discount_code (nullable string) [safe], ~ order_id (int -> varchar) [breaking]
Result: status=Halt (mixed diff, breaking change present overrides the safe addition)
```

## Dependencies
- `Schema_Management_Framework.md` (v1.0) — this component implements the decision table and pseudo logic defined there.
- `Config_Framework.md` (v1.0) — `table_name` traces to `Source_Config`; `schema_registry` is documented there as a supporting table.

## Future Extension Points
- Could add a "dry-run" mode that reports what a diff classification would be without actually being invoked mid-pipeline, useful for pre-onboarding checks on a new table.

## AI Generation Notes
Any agent generating this component must implement the all-or-nothing mixed-diff rule exactly — never allow generated code to accommodate the safe portion of a diff while ignoring the breaking portion.