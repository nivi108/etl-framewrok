# Orchestrator Agent

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** All specs (reads the full spec tree to understand system context)
**Category:** Agents

## Purpose
The parent agent that coordinates all child agents. When a user says "onboard table X" or "generate the full framework," this is the single entry point that determines which child agents to invoke, in what order, with what inputs, and how to handle failures. Also defines the common contract every child agent must follow.

---

## Common Agent Contract

Every child agent (`raw_agent`, `silver_agent`, `gold_agent`, `workflow_agent`, `testing_agent`, `deployment_agent`) follows this shared interface:

### Input Contract
Every agent receives:
```
{
    table_config: object,        # output of Read_Config_Component (or equivalent spec references)
    environment: string,         # dev/test/prod
    specs_path: string,          # path to /specs/ folder
    output_path: string          # path to write generated artifacts (per Repository_Structure.md)
}
```

### Output Contract
Every agent returns:
```
{
    status: enum(Success, Failed, Skipped),
    artifacts_generated: [{path: string, type: enum(notebook, sql, workflow, test)}],
    errors: [string],            # empty on Success
    summary: string              # human-readable one-line summary
}
```

### Behavioral Rules (All Agents)
- Read relevant specs before generating anything — never rely on training knowledge alone when a spec exists.
- Follow `Coding_Standards.md` and `Naming_Standards.md` for all generated code.
- Never hardcode table names, connection strings, or environment references.
- Always generate artifacts into the correct `output_path` subfolder per `Repository_Structure.md`.
- Return structured output (not free-text) so the orchestrator can reliably parse success/failure.

---

## Orchestrator Sequencing

### Single Table Onboarding
When onboarding one table (the common case):

```
1. deployment_agent  → generate config DDL / setup scripts (if config tables don't exist yet)
2. raw_agent         → generate Raw ingestion notebook
3. silver_agent      → generate Silver cleansing notebook
4. gold_agent        → generate Gold dimension/fact notebook (if fact_or_dimension is configured)
5. workflow_agent    → generate workflow definition wiring the above notebooks together
6. testing_agent     → generate unit tests for each notebook + integration test for the pipeline
```

### Full Framework Generation
When generating the entire framework from scratch:

```
1. deployment_agent  → generate all config table DDL, setup notebooks, secret scope documentation
2. raw_agent         → generate Raw notebooks for ALL active tables in config
3. silver_agent      → generate Silver notebooks for ALL active tables
4. gold_agent        → generate Gold notebooks for ALL tables with fact_or_dimension configured
5. workflow_agent    → generate all workflow definitions from Workflow_Config
6. testing_agent     → generate all test files
```

### Dependency Ordering Within a Step
If multiple tables are being generated in the same step, the orchestrator respects `Workflow_Config.dependency_group` and `execution_order` — dimensions are generated before facts within the same group, so that `Fact_Component`'s dimension references resolve correctly.

---

## Failure Handling

| Scenario | Orchestrator Behavior |
|---|---|
| A child agent returns `Failed` | Log the failure, skip all downstream agents for that table, continue with other tables if in batch mode |
| A child agent returns `Skipped` (e.g., no Gold config for a table) | Normal — not every table needs every agent. Continue to next agent. |
| Multiple agents fail across different tables | Collect all failures, report a consolidated summary at the end, never stop the entire batch for one table's failure |

---

## Acceptance Criteria
- [ ] Orchestrator can onboard a single table end-to-end by invoking child agents in the correct order.
- [ ] Orchestrator can generate the full framework for all active tables in batch mode.
- [ ] Every child agent follows the common input/output contract.
- [ ] Failures are isolated per table, never cascading to block unrelated tables.

## Future Extension Points
- Could add a "dry-run" mode that shows what the orchestrator *would* generate without actually writing files — useful for reviewing scope before committing.
- Could add incremental regeneration (only regenerate artifacts for tables whose config changed since last generation) rather than always full regeneration.