# Surrogate Key Component

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** Config_Framework.md (v1.0), Gold_Framework.md (v1.0)
**Category:** Components

## Purpose
Implements surrogate key generation and lookup for Gold dimensions. Exposes two distinct functions:
- `resolve_surrogate_key` — **generate-or-lookup**: returns the existing surrogate key for a known business key, or generates a new one for a previously unseen key. Used by `Dimension_Component.md` during dimension building.
- `lookup_only` — **lookup with no side effects**: returns the existing surrogate key if found, or null if not. Never generates. Used by `Fact_Component.md` during dimension foreign key resolution, since facts must never create dimension rows.

This is the component `Gold_Framework.md`'s dimension-building step and fact-building dimension-key-resolution step both depend on, but each uses the appropriate function for its role.

## Scope
Covers surrogate key generation/lookup only. Does NOT decide dimension SCD behavior (that's `SCD_Type2_Framework.md`, applied at the Gold layer per `Gold_Framework.md`) — a business key retains the same surrogate key across SCD2 versions; only the version-specific attributes change, not the surrogate key identity.

## Inputs

### `resolve_surrogate_key` (generate-or-lookup)
| Input | Type | Description |
|---|---|---|
| `business_key_value` | any | The business key value to resolve |
| `dimension_table` | string | Which Gold dimension table to look up/generate against |

### `lookup_only` (no generation)
| Input | Type | Description |
|---|---|---|
| `business_key_value` | any | The business key value to look up |
| `dimension_table` | string | Which Gold dimension table to look up against |

## Outputs
- `resolve_surrogate_key`: returns a surrogate key value (integer or UUID, per configured key strategy) — either newly generated or looked up from existing records. Never returns null.
- `lookup_only`: returns the existing surrogate key value if found, or null if the business key doesn't exist in the dimension. Never generates a new key.

## Configuration
- Key generation strategy (auto-incrementing integer vs. UUID) is a framework-wide setting, not per-table — consistency here avoids mixed key types across dimensions, which complicates fact table foreign keys.
- Default strategy: auto-incrementing integer, sequential per dimension table.

## Implementation Rules
- A business key maps to exactly one surrogate key for its entire lifetime, even across SCD2 version changes — the surrogate key identifies the "row family" (all versions of one business entity), not a specific attribute snapshot.
- Surrogate key generation must be safe under concurrent/parallel execution (per `Workflow_Orchestration_Framework.md`'s parallel execution mode) — no risk of two parallel processes generating the same key for two different business keys.
- Lookup must check existing dimension data (any version, not just current) before generating a new key — a business key already present (even if its current version differs) should never receive a second surrogate key.
- `lookup_only` must have zero side effects — it never generates a key, never modifies any table, never logs a "new key generated" event. Its sole responsibility is to answer the question "does this business key already exist?" — used specifically by `Fact_Component.md` so that facts never accidentally create dimension rows sideways.

## Error Handling
| Scenario | Behavior |
|---|---|
| `business_key_value` is null | Raise immediately — a null business key is a data quality problem that should have been caught earlier (Null_Handling_Component), not silently assigned a placeholder key here |
| Concurrent key generation collision detected | Retry the generation with a fresh key, per standard optimistic-concurrency handling |

## Logging
Logs count of new surrogate keys generated vs. existing keys reused, per batch, per dimension table.

## Return Values
Returns the resolved surrogate key value, or raises for a null business key.

## Pseudo Logic
```
FUNCTION resolve_surrogate_key(business_key_value, dimension_table):
    IF business_key_value IS NULL:
        RAISE DataQualityError("null business key reached surrogate key resolution")

    existing = SELECT surrogate_key FROM dimension_table
               WHERE business_key = business_key_value
               LIMIT 1   # any version, not just current

    IF existing EXISTS:
        RETURN existing.surrogate_key
    ELSE:
        new_key = GENERATE_NEXT_KEY(dimension_table)   # safe under concurrency
        RETURN new_key

FUNCTION lookup_only(business_key_value, dimension_table):
    IF business_key_value IS NULL:
        RETURN NULL   # facts may legitimately have null FK candidates; caller decides how to handle

    existing = SELECT surrogate_key FROM dimension_table
               WHERE business_key = business_key_value
               LIMIT 1

    RETURN existing.surrogate_key IF existing EXISTS ELSE NULL
    # Never generates a new key. Never modifies the dimension.
```

## Acceptance Criteria
- [ ] Each business key maps to exactly one surrogate key for its entire history, across all SCD2 versions.
- [ ] Concurrent parallel execution never produces duplicate surrogate keys for different business keys.
- [ ] Null business keys are rejected by `resolve_surrogate_key`, never silently assigned a key.
- [ ] `lookup_only` has zero side effects: never generates a key, never modifies data, returns null cleanly when the key isn't found.

## Example (Illustrative Only)
```
dimension_table: dim_customer
business_key_value: "CUST-1001" (first time seen)
Result: new surrogate_key = 4582 generated

Same business_key_value seen again in a later run (attributes changed)
Result: existing surrogate_key = 4582 returned (same key, new SCD2 version row created by Merge_Component)
```

## Dependencies
- `Gold_Framework.md` (v1.0) — this component implements the surrogate key step of dimension building and the dimension-key-resolution step of fact building.
- `Config_Framework.md` (v1.0) — `business_keys` sourced from `Pipeline_Config`.

## Future Extension Points
- Could support natural/composite surrogate keys (concatenated business keys) as an alternative to auto-incrementing integers, if a specific integration requires it — currently out of scope to keep key strategy uniform.

## AI Generation Notes
- Any agent generating dimension notebooks must call `resolve_surrogate_key` (generate-or-lookup) — never generate ad hoc `ROW_NUMBER()`-style keys per notebook, since that risks collisions across parallel executions and breaks the one-business-key-one-surrogate-key invariant.
- Any agent generating fact notebooks must call `lookup_only` — never `resolve_surrogate_key` — so that facts can never accidentally create dimension rows. A null result from `lookup_only` is the caller's cue to route to the "Unknown member" key (per `Fact_Component.md`), not to fall back to generation.