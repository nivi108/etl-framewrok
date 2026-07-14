# Testing Strategy

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** Project_Architecture.md (v1.0), Config_Framework.md (v1.0), Coding_Standards.md (v1.0), Naming_Standards.md (v1.0)
**Category:** Testing

## Purpose
Defines how generated code is verified — unit testing for individual Components, integration testing for end-to-end pipeline flows, and the minimum coverage requirements before any generated pipeline is considered production-ready.

## Scope
Covers testing approach, scope per test type, and pass/fail criteria. Does NOT duplicate DQ rule testing (that's `Data_Quality_Framework.md` running at runtime, not a separate test concern) — this document covers *code correctness*, not *data correctness*.

## Two Test Types

| Type | What It Verifies | When It Runs |
|---|---|---|
| Unit Tests | A single Component behaves correctly in isolation (e.g., does `Dedup_Component` correctly keep the latest row?) | After code generation, before deployment — part of CI |
| Integration Tests | An entire Raw → Silver → Gold pipeline flow works end-to-end for a sample table | After unit tests pass, before production use — part of deployment validation |

---

## Unit Testing

### What Gets Unit Tested
Every Component that contains logic (not just pass-through wiring) needs at least one unit test:

| Component | Key Behavior to Test |
|---|---|
| `Read_Config_Component` | Returns correct joined config; raises `ConfigNotFoundError` for missing tables; respects `active_flag` |
| `Full_Load_Component` | Returns full DataFrame with `write_mode=overwrite`; warns on large tables |
| `Incremental_Load_Component` | Filters correctly by watermark; handles first-run (null watermark) |
| `CDC_Component` | Tags every row with `_operation_type`; preserves deletes; raises `CDCGapError` on retention gap |
| `Watermark_Component` | Returns correct last value; rejects regression; handles first-run null |
| `Schema_Validation_Component` | Passes unchanged schema; accommodates additive changes; halts on breaking changes; handles first-run |
| `Dedup_Component` | Resolves intra-batch duplicates; drops stale cross-batch re-deliveries; flags null tie-break |
| `Null_Handling_Component` | Applies defaults; rejects non-nullable nulls; passes nullable nulls |
| `Type_Casting_Component` | Casts correctly; rejects failed casts; passes unconfigured columns unchanged |
| `Hashing_Component` | Produces identical hash for identical data regardless of column order; handles nulls consistently |
| `Merge_Component` | Simple upsert works correctly; SCD2 expire+insert produces exactly one current row per key; idempotent on re-run |
| `Surrogate_Key_Component` | Generates new key for new business key; returns existing key for known key; `lookup_only` never generates |
| `Dimension_Component` | Ensures Unknown member exists; correct merge mode selected by `scd_type` |
| `Fact_Component` | Resolves all dimension FKs; falls back to Unknown member; computes measures correctly |
| `Audit_Component` | Reconciliation formula correct per layer; writes record even on partial failure |

### Unit Test Structure
Each unit test follows a fixed pattern:

1. **Arrange** — create a small, in-memory test DataFrame and a mock config object (no real Delta tables, no real SQL Server connection)
2. **Act** — call the Component function with the test inputs
3. **Assert** — verify the output matches expected results exactly

### Unit Test Naming
Per `Naming_Standards.md` conventions: `test_{component}_{scenario}` (e.g., `test_dedup_intra_batch_keeps_latest`, `test_watermark_rejects_regression`).

### Unit Test Location
All unit tests live in `src/tests/unit/`, one test file per Component: `test_{component_name}.py`.

---

## Integration Testing

### What Gets Integration Tested
A complete pipeline for a **sample table** (not a real production table) — verifying that Raw → Silver → Gold flows correctly end-to-end with all Components wired together.

### Integration Test Approach

| Step | What Happens |
|---|---|
| 1. Setup | Create a temporary test schema/catalog in Databricks; populate config tables with sample-table entries; seed a small source dataset |
| 2. Run Raw | Execute the Raw ingestion notebook against the sample source; verify Raw table exists with correct audit columns and row count |
| 3. Run Silver | Execute the Silver notebook; verify cleansing, dedup, and rejected records behave correctly |
| 4. Run Gold | Execute the Gold dimension and/or fact notebook; verify surrogate keys, dimension FK resolution, and measures |
| 5. Verify Audit | Confirm audit records exist for each layer with `reconciliation_status = Match` |
| 6. Verify Logging | Confirm execution logs exist with terminal status for each notebook |
| 7. Verify Idempotency | Re-run the entire pipeline with the same data and verify zero net changes (no duplicates, no version inflation) |
| 8. Teardown | Drop temporary test schema/tables |

### Integration Test Location
All integration tests live in `src/tests/integration/`, named `test_pipeline_{domain}.py` (e.g., `test_pipeline_sales.py`).

---

## Test Coverage Requirements

| Requirement | Minimum |
|---|---|
| Unit test coverage | Every Component listed above must have tests covering its happy path + at least one failure/edge case |
| Integration test coverage | At least one full pipeline (Raw → Silver → Gold) for one sample table must pass before any production deployment |
| Idempotency test | Required as part of integration — every pipeline must be proven re-runnable without side effects |

### What Does NOT Need Separate Tests
- DQ rule evaluation — already validated at runtime by `Data_Quality_Framework.md`; no separate test layer needed.
- Notification delivery — tested manually or via integration; unit testing email/webhook delivery has low value.
- Profiling — non-blocking by design; a failed profile never impacts correctness.

---

## Acceptance Criteria
- [ ] Every logic-bearing Component has at least one unit test file.
- [ ] At least one integration test covers the full Raw → Silver → Gold flow.
- [ ] Idempotency is explicitly verified (re-run produces zero net changes).
- [ ] All tests pass before any generated code is deployed to production.

## Dependencies
- `Project_Architecture.md` (v1.0) — defines the layers integration tests must cover.
- `Config_Framework.md` (v1.0) — integration tests need sample config entries.
- `Coding_Standards.md` (v1.0) — test code follows the same coding standards as production code.
- `Naming_Standards.md` (v1.0) — test file naming follows established patterns.

## Future Extension Points
- Could add performance/load testing (verifying pipeline handles expected data volumes within acceptable time) once production-scale volumes are known.
- Could add regression testing (re-running tests after a spec change to verify downstream code still works) as part of the `Governance/Spec_Versioning.md` process.

## AI Generation Notes
Any agent generating code must also generate corresponding unit tests. A Component without a test file is considered incomplete — the `testing_agent` (defined in `/agents/`) is responsible for generating test files, but all agents share the obligation to produce testable code (clear inputs/outputs, no hidden state, single responsibility).