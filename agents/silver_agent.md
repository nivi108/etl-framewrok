# Silver Agent

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** orchestrator_agent.md (common contract), Silver_Framework.md, SCD_Type2_Framework.md, Data_Quality_Framework.md, Config_Framework.md, Coding_Standards.md, Naming_Standards.md
**Category:** Agents

## Purpose
Generates Silver layer cleansing notebooks — one per active table. Each notebook implements the eight-stage Silver processing pipeline (per `Silver_Framework.md`), including SCD Type 2 when configured (per `SCD_Type2_Framework.md`).

## What This Agent Reads
| Source | What It Extracts |
|---|---|
| `Silver_Framework.md` | Eight-stage pipeline ordering, rejected records handling |
| `SCD_Type2_Framework.md` | SCD2 merge logic, hash comparison, versioning |
| `Data_Quality_Framework.md` | Rule structure, severity-based branching |
| `Config_Framework.md` | Field definitions for `merge_keys`, `business_keys`, `scd_type`, `null_rules`, `column_type_map`, `dq_rules` |
| `Coding_Standards.md`, `Naming_Standards.md` | Code style and artifact naming |
| `Pipeline_Config` (runtime) | Per-table Silver-layer row |
| `Validation_Config` (runtime) | Per-table DQ rules and error handling strategy |

## What This Agent Generates
| Artifact | Location | Naming |
|---|---|---|
| One notebook per table | `src/notebooks/silver/clean_{table_name}.py` | Per `Naming_Standards.md` |

## Generation Rules
- Notebook must implement all eight stages in exact order: schema validation → null handling → type casting → dedup → standardization → business rules → DQ rules → merge.
- Must use shared Components (`Null_Handling_Component`, `Type_Casting_Component`, `Dedup_Component`, `Hashing_Component`, `Merge_Component`) — never inline custom logic.
- If `scd_type = 2`, stage 8 merge must use `Merge_Component` in `scd2` mode.
- If `scd_type = 1` or `None`, stage 8 merge must use `simple_upsert` mode.
- Must route rejected records to `silver_{table_name}_rejected` table.
- DQ rule severity (`blocking`/`non_blocking`) must be read from config, never assumed.
- Must call `Profiling_Component` as a non-blocking side step.
- Must call `Logging_Component` at start/end and `Audit_Component` at end (in try/finally).

## Acceptance Criteria
- [ ] All eight stages present in correct order.
- [ ] SCD2 merge logic matches `SCD_Type2_Framework.md` exactly when `scd_type = 2`.
- [ ] Rejected records always land in the rejected table, never silently dropped.
- [ ] DQ severity drives behavior, not hardcoded assumptions.