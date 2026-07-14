# Workflow Agent

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** orchestrator_agent.md (common contract), Workflow_Orchestration_Framework.md, Config_Framework.md, Naming_Standards.md, CICD_Pipeline_Deployment.md
**Category:** Agents

## Purpose
Generates Databricks workflow/job definitions from `Workflow_Config` — wiring generated notebooks into scheduled, dependency-ordered, parallel-aware workflows. Produces the JSON/configuration artifacts that Databricks uses to create and schedule jobs.

## What This Agent Reads
| Source | What It Extracts |
|---|---|
| `Workflow_Orchestration_Framework.md` | Execution modes, dependency graph resolution, failure recovery, restart logic |
| `Config_Framework.md` | `Workflow_Config` field definitions |
| `Naming_Standards.md` | Workflow naming pattern (`wf_{domain}_{frequency}`) |
| `CICD_Pipeline_Deployment.md` | Environment parameter must be passed to every notebook task |
| `Workflow_Config` (runtime) | Per-table: `workflow_name`, `notebook_name`, `schedule`, `dependency_group`, `execution_order`, `parallel_group`, `retry_count`, `timeout_minutes` |

## What This Agent Generates
| Artifact | Location | Naming |
|---|---|---|
| One workflow definition per `workflow_name` | `src/workflows/{workflow_name}.json` | Per `Naming_Standards.md` |

## Generation Rules
- Group tables by `workflow_name`, then resolve execution order from `dependency_group`, `execution_order`, and `parallel_group`.
- Tables with the same `execution_order` and `parallel_group` must be wired as parallel tasks in the workflow.
- Every notebook task must receive the `environment` parameter.
- `retry_count` and `timeout_minutes` from config must be applied to each task.
- Dimension tasks must precede fact tasks within the same dependency group — validate this from config before generating, and raise an error if the config violates this constraint.
- Must not run a workflow concurrently with itself (per `Workflow_Orchestration_Framework.md`) — configure `max_concurrent_runs = 1`.

## Acceptance Criteria
- [ ] Workflow correctly sequences parallel and sequential tasks per config.
- [ ] Every notebook task receives the `environment` parameter.
- [ ] Dimension-before-fact ordering is validated, not just assumed.
- [ ] `max_concurrent_runs = 1` is set on every workflow.