# Testing Agent

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** orchestrator_agent.md (common contract), Testing_Strategy.md, Coding_Standards.md, Naming_Standards.md
**Category:** Agents

## Purpose
Generates unit test files for each Component and integration test files for end-to-end pipeline validation, per `Testing_Strategy.md`.

## What This Agent Reads
| Source | What It Extracts |
|---|---|
| `Testing_Strategy.md` | Unit test structure (arrange/act/assert), integration test steps, coverage requirements |
| Component specs (all) | Key behaviors to test per Component (the table in Testing_Strategy.md) |
| `Coding_Standards.md`, `Naming_Standards.md` | Test code style and file naming (`test_{component}_{scenario}.py`) |

## What This Agent Generates
| Artifact | Location | Naming |
|---|---|---|
| Unit test files | `src/tests/unit/test_{component_name}.py` | One file per Component |
| Integration test files | `src/tests/integration/test_pipeline_{domain}.py` | One file per dependency group |

## Generation Rules
- Every logic-bearing Component gets at least one unit test covering the happy path and one covering a failure/edge case.
- Unit tests use in-memory DataFrames and mock configs — no real Delta tables or SQL Server connections.
- Integration tests follow the seven-step flow from `Testing_Strategy.md` (setup → Raw → Silver → Gold → verify audit → verify logging → verify idempotency → teardown).
- Integration tests use a temporary test schema, created at setup and dropped at teardown.
- Test code follows the same `Coding_Standards.md` as production code.

## Acceptance Criteria
- [ ] Every Component listed in `Testing_Strategy.md`'s coverage table has a corresponding test file.
- [ ] At least one integration test covers a full Raw → Silver → Gold flow.
- [ ] Idempotency re-run is included in integration tests.