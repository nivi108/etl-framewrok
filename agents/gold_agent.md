# Gold Agent

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** orchestrator_agent.md (common contract), Gold_Framework.md, SCD_Type2_Framework.md, Config_Framework.md, Coding_Standards.md, Naming_Standards.md
**Category:** Agents

## Purpose
Generates Gold layer notebooks — dimension builders, fact builders, and aggregate builders — based on `Pipeline_Config.fact_or_dimension` for each table. One notebook per Gold object.

## What This Agent Reads
| Source | What It Extracts |
|---|---|
| `Gold_Framework.md` | Dimension/fact/aggregate building rules, dependency ordering, key resolution |
| `SCD_Type2_Framework.md` | SCD2 logic for Gold dimensions when `scd_type = 2` |
| `Config_Framework.md` | Field definitions for `fact_or_dimension`, `business_keys`, `merge_keys`, `measure_definitions`, `fact_write_mode`, `aggregate_definition` |
| `Coding_Standards.md`, `Naming_Standards.md` | Code style and artifact naming |
| `Pipeline_Config` (runtime) | Per-table Gold-layer row |

## What This Agent Generates
| Artifact | Location | Naming |
|---|---|---|
| Dimension notebook | `src/notebooks/gold/build_dim_{entity}.py` | Per `Naming_Standards.md` |
| Fact notebook | `src/notebooks/gold/build_fact_{business_process}.py` | Per `Naming_Standards.md` |
| Aggregate notebook | `src/notebooks/gold/build_agg_{summary}.py` | Per `Naming_Standards.md` |

## Generation Rules
- Must use `Dimension_Component` for dimensions — never inline surrogate key or SCD logic.
- Must use `Fact_Component` for facts — always `lookup_only` for dimension key resolution, fall back to Unknown member, never generate new dimension rows from fact logic.
- Must use `Aggregation_Component` for aggregates — respect `refresh_mode` (incremental vs. full_recompute).
- If `fact_or_dimension = None` for a table, this agent returns `Skipped` — not every table needs a Gold object.
- Dimension notebooks must be generated before fact notebooks within the same dependency group (per `Workflow_Orchestration_Framework.md`'s ordering constraint).
- Must call `Logging_Component` at start/end and `Audit_Component` at end (in try/finally).

## Acceptance Criteria
- [ ] No fact notebook permits null dimension foreign keys.
- [ ] Dimension notebooks ensure Unknown member row exists.
- [ ] Aggregate notebooks are always derivable from facts — never a divergent source.
- [ ] Agent returns `Skipped` for tables without Gold config, not an error.