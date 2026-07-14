# Repository Structure

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** Project_Architecture.md (v1.0), Medallion_Architecture.md (v1.0)
**Category:** Architecture

## Purpose
Defines the physical folder structure for the entire project repository — both the specification tree (already in use) and the code/notebook tree that will be generated once we move past the spec-writing phase. This is the map an AI agent uses to know *where* to place generated artifacts, matching the layers and contracts defined in `Project_Architecture.md` and `Medallion_Architecture.md`.

## Scope
Covers folder layout only — not file content (that's each individual spec/component doc) and not naming conventions for files within folders (that's `Standards/Naming_Standards.md`).

## Full Repository Structure

```
etl-framework/
│
├── specs/                          # Design-time markdown specifications
│   ├── Architecture/
│   │   ├── Project_Architecture.md
│   │   ├── Medallion_Architecture.md
│   │   └── Repository_Structure.md
│   ├── Frameworks/
│   │   ├── Config_Framework.md
│   │   ├── Schema_Management_Framework.md
│   │   ├── Ingestion_Framework.md
│   │   ├── Raw_Framework.md
│   │   ├── Silver_Framework.md
│   │   ├── SCD_Type2_Framework.md
│   │   ├── Gold_Framework.md
│   │   ├── Logging_Framework.md
│   │   ├── Audit_Framework.md
│   │   ├── Data_Quality_Framework.md
│   │   ├── Error_Handling_Framework.md
│   │   └── Workflow_Orchestration_Framework.md
│   ├── Components/
│   │   ├── (20 component files — see Components/ listing)
│   ├── Standards/
│   │   ├── Coding_Standards.md
│   │   └── Naming_Standards.md
│   ├── Testing/
│   │   └── Testing_Strategy.md
│   ├── Deployment/
│   │   ├── CICD_Pipeline_Deployment.md
│   │   └── Secrets_Management.md
│   ├── Governance/
│   │   ├── Dependency_Manifest.md
│   │   └── Spec_Versioning.md
│   └── README.md
│
├── agents/                         # Agent behavior specs (what each agent reads/generates)
│   ├── orchestrator_agent.md
│   ├── raw_agent.md
│   ├── silver_agent.md
│   ├── gold_agent.md
│   ├── workflow_agent.md
│   ├── testing_agent.md
│   └── deployment_agent.md
│
├── skills/                         # Step-by-step generation recipes invoked by agents
│   ├── create_raw.skill.md
│   ├── create_silver.skill.md
│   ├── create_dimension.skill.md
│   ├── create_fact.skill.md
│   └── create_workflow.skill.md
│
└── src/                             # NOT YET CREATED — generated code lands here once we begin code generation
    ├── notebooks/
    │   ├── raw/
    │   ├── silver/
    │   ├── gold/
    │   └── setup/
    ├── sql/
    │   └── ddl/
    ├── workflows/
    └── tests/
        ├── unit/
        └── integration/
```

## Folder Responsibility Table

| Folder | Contains | Created When | Consumed By |
|---|---|---|---|
| `specs/` | Design-time markdown specifications | Now (complete) | AI agents, during code generation |
| `agents/` | Behavior contracts for each generation agent | Now (complete) | The agents themselves, when invoked |
| `skills/` | Step-by-step generation recipes | Now (complete) | Agents, as sub-routines |
| `src/` | Actual generated PySpark/SQL/workflow code | Only once we reach code-generation phase | Databricks Repos (synced) |

## What Databricks Actually Needs to See

Per our earlier discussion, Databricks Repos should sync only the `src/` folder (or a subset of it) — not the entire `specs/`, `agents/`, `skills/`, `templates/` tree. Those are development-time artifacts.

| Synced to Databricks? | Folder |
|---|---|
| No | `specs/`, `agents/`, `skills/` |
| Yes | `src/notebooks/`, `src/sql/`, `src/workflows/` |
| Yes (for CI) | `src/tests/` |

## Best Practices
- Keep `specs/` and `src/` in the same Git repo (single source of truth, shared version history) but treat them as logically separate concerns.
- Never let generated code in `src/` be hand-edited without the corresponding spec being updated first — this is the "spec drift" risk from `Governance/Spec_Versioning.md` applied at the code level, not just the spec level.

## Validation Rules
- No generated notebook may be placed outside the `src/notebooks/{raw|silver|gold}/` structure.
- No spec file may be placed outside its designated `specs/` subfolder.

## Pseudo Logic
```
FUNCTION resolve_output_path(artifact_type, layer):
    IF artifact_type == "notebook":
        RETURN f"src/notebooks/{layer}/"
    IF artifact_type == "ddl":
        RETURN "src/sql/ddl/"
    IF artifact_type == "workflow":
        RETURN "src/workflows/"
    IF artifact_type == "test":
        RETURN f"src/tests/{test_category}/"
```

## Acceptance Criteria
- [ ] Every category in `specs/` matches a corresponding folder that actually exists in the repo.
- [ ] `src/` folder is documented here even though not yet populated, so agents know where to write once code generation starts.
- [ ] No ambiguity about what syncs to Databricks vs. what stays purely in Git/VS Code.

## Example
```
New table "Inventory" onboarded →
  generated Raw notebook  → src/notebooks/raw/load_inventory.py
  generated Silver notebook → src/notebooks/silver/clean_inventory.py
  generated Gold notebook  → src/notebooks/gold/dim_inventory.py (if dimension)
  generated DDL            → src/sql/ddl/inventory_config.sql
  generated workflow def   → src/workflows/inventory_workflow.json
```

## Dependencies
- `Project_Architecture.md` (v1.0) — defines the layers this structure organizes around.
- `Medallion_Architecture.md` (v1.0) — defines the Raw/Silver/Gold contract reflected in the `src/notebooks/` subfolders.

## Future Extension Points
- If a streaming ingestion path is added (per the Future Extension Point noted in `Project_Architecture.md`), a `src/notebooks/streaming/` folder would be added here.
- If multiple source systems are eventually supported, `src/notebooks/raw/` may need per-source subfolders (e.g., `raw/sqlserver/`, `raw/salesforce/`).

## AI Generation Notes
Every agent must resolve its output path using the Folder Responsibility Table and Pseudo Logic above — never infer or invent a path. If an agent needs to write an artifact type not covered here, that's a signal this spec needs updating before proceeding (flag to user, per `Spec_Versioning.md`).