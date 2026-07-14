# Hashing Component

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** Config_Framework.md (v1.0), SCD_Type2_Framework.md (v1.0)
**Category:** Components

## Purpose
Implements deterministic row hashing (`_row_hash`) used by `SCD_Type2_Framework.md` for change detection — comparing an incoming row's computed hash against the current Silver row's stored hash to determine whether a business entity's attributes have changed.

## Scope
Covers hash computation only. Does NOT decide what happens when hashes differ (that's `SCD_Type2_Framework.md`'s merge logic) — this component's sole job is producing a consistent, comparable hash value.

## Inputs
| Input | Type | Description |
|---|---|---|
| `row` | struct | The row to hash |
| `tracked_columns` | array[string] | The columns whose values should be included in the hash — explicitly excludes keys and audit columns |

## Outputs
A single hash string (`_row_hash`) representing the combined value of all `tracked_columns` for that row.

## Configuration
`tracked_columns` is derived as: all columns in the table EXCEPT `business_keys`, `merge_keys`, surrogate keys, and any audit/SCD columns (`_effective_date`, `_expiry_date`, `_is_current`, `_version`, `_row_hash` itself, etc.) — this exclusion list must be applied consistently, never manually curated per table in a way that could drift between tables.

## Implementation Rules
- Column ordering used for hash computation must be deterministic (e.g., alphabetical by column name) — never dependent on the DataFrame's incidental column order, which could vary between runs and produce different hashes for identical data.
- Null values must be hashed using a consistent sentinel representation (e.g., treat null as a fixed placeholder string) — inconsistent null handling (sometimes skipping the column, sometimes using empty string) would break hash comparability.
- The hash function itself (e.g., SHA-256 over the concatenated, ordered column values) must be the same function used across every table — no per-table hash function customization, since that would break the ability to reason about hash behavior generically.

## Error Handling
| Scenario | Behavior |
|---|---|
| `tracked_columns` list is empty (misconfiguration — table has only keys/audit columns) | Raise a config error — a table with nothing to track has no meaningful SCD2 use case, this indicates a setup mistake |

## Logging
Minimal — hashing is a pure, deterministic function with no meaningful per-row logging needed. Logs only aggregate stats if requested (e.g., count of rows hashed per batch), not per-row hash values (which would be meaningless in a log without full row context anyway).

## Return Values
Returns the computed hash string for a row, or a config error if `tracked_columns` is invalid.

## Pseudo Logic
```
FUNCTION compute_row_hash(row, tracked_columns):
    IF tracked_columns IS EMPTY:
        RAISE ConfigError("no trackable columns for hashing")

    sorted_columns = SORT_ALPHABETICALLY(tracked_columns)
    values = []
    FOR each column in sorted_columns:
        value = row[column]
        IF value IS NULL:
            values.append("__NULL__")   # consistent sentinel
        ELSE:
            values.append(TO_STRING(value))

    concatenated = JOIN(values, delimiter="|")
    RETURN SHA256(concatenated)
```

## Acceptance Criteria
- [ ] Identical row content always produces an identical hash, regardless of DataFrame column ordering.
- [ ] Null values are hashed consistently using the same sentinel every time.
- [ ] The same hash function is used across every table in the framework — no per-table variation.

## Example (Illustrative Only)
```
tracked_columns: [address, phone_number]
row: {address: "456 Oak Ave", phone_number: null}
Sorted, sentineled: ["456 Oak Ave", "__NULL__"]
Concatenated: "456 Oak Ave|__NULL__"
_row_hash: SHA256("456 Oak Ave|__NULL__") = "a3f9..."
```

## Dependencies
- `SCD_Type2_Framework.md` (v1.0) — this component implements the hash comparison mechanism referenced there.
- `Config_Framework.md` (v1.0) — `tracked_columns` is derived from `business_keys`/`merge_keys` already defined in `Pipeline_Config`, by exclusion.

## Future Extension Points
- Could support configurable hash algorithms (e.g., MD5 for speed if collision risk is deemed acceptable for a given use case) — currently fixed to SHA-256 for consistency and security margin.

## AI Generation Notes
Any agent generating this component must sort columns deterministically before hashing and must never let hash computation logic vary between tables — the entire value of this component is that every table's `_row_hash` behaves identically and predictably.