# ETL Framework — Specification-Driven Development

## What This Is
A complete set of markdown specifications for an enterprise-grade, metadata-driven ETL framework on Azure Databricks. These specs are designed to be read by AI agents that generate the actual PySpark notebooks, SQL DDL, workflow definitions, and tests — no human coding decisions required.

## Architecture at a Glance
- **Source:** SQL Server
- **Destination:** Azure Databricks (Delta Lake)
- **Pattern:** Medallion Architecture (Raw → Silver → Gold)
- **Principle:** Everything is config-driven — no hardcoded table names, notebook names, or workflows

## How This Repo Is Organized

```
specs/           → What to build and how (the specifications)
agents/          → Which agent generates what, and in what order
skills/          → Step-by-step recipes for generating each artifact type
src/             → Generated code (notebooks, SQL, workflows, tests) — created by agents, not by hand
```

## Reading Order
If you're an AI agent or a human trying to understand the framework, read in this order:

1. **Start here:** `specs/Architecture/Project_Architecture.md` — the system overview
2. **Layer contracts:** `specs/Architecture/Medallion_Architecture.md` — what each layer must satisfy
3. **Config model:** `specs/Frameworks/Config_Framework.md` — the metadata model everything else reads from
4. **Data flow:** Ingestion → Raw → Silver → SCD2 → Gold Frameworks (in that order)
5. **Reliability:** Logging → Audit → Data Quality → Error Handling Frameworks
6. **Orchestration:** `specs/Frameworks/Workflow_Orchestration_Framework.md`
7. **Components:** Individual reusable modules (20 files, each narrow in scope)
8. **Standards:** Coding and naming conventions
9. **Governance:** `specs/Governance/Dependency_Manifest.md` — tracks all cross-file dependencies

## Key Design Decisions
- **Config tables, not YAML:** Runtime metadata lives in Delta tables, not files. Specs define behavior; config tables define which objects exist.
- **Specs define what, skills define how:** Specs are the rules; skills are the step-by-step generation recipes agents follow mechanically.
- **Four core config tables:** `Source_Config`, `Pipeline_Config`, `Workflow_Config`, `Validation_Config` — plus supporting tables (`schema_registry`, `watermark_registry`).
- **No separate templates folder:** Skills absorbed the role of templates by including exact notebook skeletons in their step-by-step instructions.
- **No separate config/ docs folder:** `Config_Framework.md` is the single source of truth for all config table schemas.

## Onboarding a New Table
Once the framework is generated and running, adding a new source table is a config-only exercise:
1. Add rows to `Source_Config`, `Pipeline_Config`, `Workflow_Config`, `Validation_Config`.
2. Run the orchestrator agent to generate notebooks/workflows for the new table.
3. Run tests.
4. Enable the workflow schedule.

No spec changes, no code changes — unless the new table needs behavior the framework doesn't yet support (which gets tracked in `Governance/Spec_Versioning.md`'s exceptions log).

## File Count Summary
| Category | Files |
|---|---|
| Architecture | 3 |
| Frameworks | 12 |
| Components | 20 |
| Standards | 2 |
| Testing | 1 |
| Deployment | 2 |
| Governance | 2 |
| Agents | 7 |
| Skills | 5 |
| **Total** | **54** |