# Raw Agent

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** orchestrator_agent.md (common contract), Ingestion_Framework.md, Raw_Framework.md, Schema_Management_Framework.md, Config_Framework.md, Coding_Standards.md, Naming_Standards.md
**Category:** Agents

## Purpose
Generates Raw layer ingestion notebooks — one per active table in `Source_Config`. Each notebook implements the Full/Incremental/CDC load path (per `Ingestion_Framework.md`) and the Raw write/partitioning behavior (per `Raw_Framework.md`).

## What This Agent Reads
| Source | What It Extracts |
|---|---|
| `Ingestion_Framework.md` | Load type process flows, recovery/restart logic |
| `Raw_Framework.md` | Table naming, partitioning, audit columns, duplicate prevention |
| `Schema_Management_Framework.md` | Schema validation gate before write |
| `Config_Framework.md` | Field definitions to know which config fields the notebook must read |
| `Coding_Standards.md`, `Naming_Standards.md` | Code style and artifact naming |
| `Source_Config` (runtime) | Per-table: `load_type`, `primary_keys`, `incremental_column`, `watermark_column`, `cdc_enabled` |
| `Pipeline_Config` (runtime) | Per-table: `partition_columns`, `retention_policy` |

## What This Agent Generates
| Artifact | Location | Naming |
|---|---|---|
| One notebook per table | `src/notebooks/raw/load_{table_name}.py` | Per `Naming_Standards.md` |

## Generation Rules
- Notebook must accept `environment` parameter (per `CICD_Pipeline_Deployment.md`).
- Notebook must call `Read_Config_Component` first, then branch on `load_type`.
- Must call `Schema_Validation_Component` before every write.
- Must call `Logging_Component` at start/end and `Audit_Component` at end.
- Must implement `_load_id` duplicate check for Incremental/CDC (per `Raw_Framework.md`).
- Must never advance watermark before confirming write success.
- Must use `Partitioning_Component` for write partitioning.

## Acceptance Criteria
- [ ] Generated notebook correctly implements the load type declared in config, not a hardcoded assumption.
- [ ] All required audit columns (`_ingested_at`, `_source_system`, `_load_id`, `_watermark_value`) are present.
- [ ] Logging and audit calls are present in try/finally blocks.