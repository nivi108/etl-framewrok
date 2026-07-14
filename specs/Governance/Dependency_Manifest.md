# Dependency Manifest

**Version:** 1.0
**Last Modified:** 2026-07-14
**Depends On:** None (root document)
**Category:** Governance

## Purpose
This document is the single source of truth for tracking dependencies between all specification files in the `/specs` folder. It exists to prevent spec drift — when one file changes, this manifest tells you exactly which other files need to be reviewed.

## Scope
Covers all files under `specs/`. Does not cover `agents/`, `skills/`, or `src/` — those are consumers of specs, not specs themselves.

## How to Use This Manifest
1. Every spec file, when created or modified, must have an entry below.
2. The `Depends On` column lists every other spec file it references, along with the version.
3. When you modify a file, search this manifest for every entry where your file appears in someone else's `Depends On` column — those are the files that may now be stale.
4. After reviewing/updating a dependent file, bump its version in both the file's own header and in this manifest.

## Dependency Direction Rule
Dependencies flow one way only:
`Architecture → Frameworks → Components → Standards`

A file may only depend on files to its left in this chain (or within the same category, on a file already listed above it). No file may depend on something later in the chain.

## Manifest Table

### Architecture

| Spec File | Depends On | Version | Status |
|---|---|---|---|
| `Project_Architecture.md` | None | 1.0 | Active |
| `Medallion_Architecture.md` | Project_Architecture (v1.0) | 1.0 | Active |
| `Repository_Structure.md` | Project_Architecture (v1.0), Medallion_Architecture (v1.0) | 1.0 | Active |

### Frameworks

| Spec File | Depends On | Version | Status |
|---|---|---|---|
| `Config_Framework.md` | Project_Architecture (v1.0), Medallion_Architecture (v1.0), Repository_Structure (v1.0) | 1.0 | Active |
| `Schema_Management_Framework.md` | Project_Architecture (v1.0), Medallion_Architecture (v1.0), Config_Framework (v1.0) | 1.0 | Active |
| `Ingestion_Framework.md` | Project_Architecture (v1.0), Medallion_Architecture (v1.0), Config_Framework (v1.0), Schema_Management_Framework (v1.0) | 1.0 | Active |
| `Raw_Framework.md` | Medallion_Architecture (v1.0), Config_Framework (v1.0), Schema_Management_Framework (v1.0), Ingestion_Framework (v1.0) | 1.0 | Active |
| `Silver_Framework.md` | Medallion_Architecture (v1.0), Config_Framework (v1.0), Schema_Management_Framework (v1.0), Raw_Framework (v1.0) | 1.0 | Active |
| `SCD_Type2_Framework.md` | Medallion_Architecture (v1.0), Config_Framework (v1.0), Silver_Framework (v1.0) | 1.0 | Active |
| `Gold_Framework.md` | Medallion_Architecture (v1.0), Config_Framework (v1.0), Silver_Framework (v1.0), SCD_Type2_Framework (v1.0) | 1.0 | Active |
| `Logging_Framework.md` | Project_Architecture (v1.0), Config_Framework (v1.0), Ingestion_Framework (v1.0), Raw_Framework (v1.0), Silver_Framework (v1.0), Gold_Framework (v1.0) | 1.0 | Active |
| `Audit_Framework.md` | Config_Framework (v1.0), Ingestion_Framework (v1.0), Raw_Framework (v1.0), Silver_Framework (v1.0), Gold_Framework (v1.0), Logging_Framework (v1.0) | 1.0 | Active |
| `Data_Quality_Framework.md` | Config_Framework (v1.0), Silver_Framework (v1.0), Audit_Framework (v1.0) | 1.0 | Active |
| `Error_Handling_Framework.md` | Config_Framework (v1.0), Data_Quality_Framework (v1.0), Logging_Framework (v1.0), Audit_Framework (v1.0) | 1.0 | Active |
| `Workflow_Orchestration_Framework.md` | Config_Framework (v1.0), Medallion_Architecture (v1.0), Gold_Framework (v1.0), Error_Handling_Framework (v1.0), Logging_Framework (v1.0), Audit_Framework (v1.0) | 1.0 | Active |

### Components

| Spec File | Depends On | Version | Status |
|---|---|---|---|
| `Read_Config_Component.md` | Config_Framework (v1.0) | 1.0 | Active |
| `SQL_Server_Reader_Component.md` | Config_Framework (v1.0), Ingestion_Framework (v1.0), Read_Config_Component (v1.0) | 1.0 | Active |
| `Full_Load_Component.md` | Config_Framework (v1.0), Ingestion_Framework (v1.0), SQL_Server_Reader_Component (v1.0), Read_Config_Component (v1.0) | 1.0 | Active |
| `Incremental_Load_Component.md` | Config_Framework (v1.0), Ingestion_Framework (v1.0), SQL_Server_Reader_Component (v1.0), Read_Config_Component (v1.0), Watermark_Component (v1.0) | 1.0 | Active |
| `CDC_Component.md` | Config_Framework (v1.0), Ingestion_Framework (v1.0), SQL_Server_Reader_Component (v1.0), Read_Config_Component (v1.0), Watermark_Component (v1.0) | 1.0 | Active |
| `Watermark_Component.md` | Config_Framework (v1.0), Ingestion_Framework (v1.0), Audit_Framework (v1.0) | 1.0 | Active |
| `Schema_Validation_Component.md` | Config_Framework (v1.0), Schema_Management_Framework (v1.0) | 1.0 | Active |
| `Dedup_Component.md` | Config_Framework (v1.0), Silver_Framework (v1.0) | 1.0 | Active |
| `Null_Handling_Component.md` | Config_Framework (v1.0), Silver_Framework (v1.0) | 1.0 | Active |
| `Type_Casting_Component.md` | Config_Framework (v1.0), Silver_Framework (v1.0) | 1.0 | Active |
| `Hashing_Component.md` | Config_Framework (v1.0), SCD_Type2_Framework (v1.0) | 1.0 | Active |
| `Merge_Component.md` | Config_Framework (v1.0), Silver_Framework (v1.0), SCD_Type2_Framework (v1.0), Gold_Framework (v1.0) | 1.0 | Active |
| `Surrogate_Key_Component.md` | Config_Framework (v1.0), Gold_Framework (v1.0) | 1.0 | Active |
| `Dimension_Component.md` | Config_Framework (v1.0), Gold_Framework (v1.0), Surrogate_Key_Component (v1.0), Merge_Component (v1.0), SCD_Type2_Framework (v1.0) | 1.0 | Active |
| `Fact_Component.md` | Config_Framework (v1.0), Gold_Framework (v1.0), Surrogate_Key_Component (v1.0), Merge_Component (v1.0), Dimension_Component (v1.0) | 1.0 | Active |
| `Aggregation_Component.md` | Config_Framework (v1.0), Gold_Framework (v1.0), Merge_Component (v1.0), Fact_Component (v1.0) | 1.0 | Active |
| `Partitioning_Component.md` | Config_Framework (v1.0), Raw_Framework (v1.0), Gold_Framework (v1.0) | 1.0 | Active |
| `Profiling_Component.md` | Config_Framework (v1.0), Silver_Framework (v1.0), Logging_Framework (v1.0) | 1.0 | Active |
| `Audit_Component.md` | Config_Framework (v1.0), Audit_Framework (v1.0), Logging_Framework (v1.0) | 1.0 | Active |
| `Logging_Component.md` | Config_Framework (v1.0), Logging_Framework (v1.0) | 1.0 | Active |
| `Notification_Component.md` | Config_Framework (v1.0), Error_Handling_Framework (v1.0), Audit_Framework (v1.0), Logging_Framework (v1.0) | 1.0 | Active |

### Standards

| Spec File | Depends On | Version | Status |
|---|---|---|---|
| `Coding_Standards.md` | Project_Architecture (v1.0), Repository_Structure (v1.0) | 1.0 | Active |
| `Naming_Standards.md` | Project_Architecture (v1.0), Medallion_Architecture (v1.0), Repository_Structure (v1.0) | 1.0 | Active |

### Testing

| Spec File | Depends On | Version | Status |
|---|---|---|---|
| `Testing_Strategy.md` | Project_Architecture (v1.0), Config_Framework (v1.0), Coding_Standards (v1.0), Naming_Standards (v1.0) | 1.0 | Active |

### Deployment

| Spec File | Depends On | Version | Status |
|---|---|---|---|
| `CICD_Pipeline_Deployment.md` | Project_Architecture (v1.0), Repository_Structure (v1.0), Testing_Strategy (v1.0), Coding_Standards (v1.0) | 1.0 | Active |
| `Secrets_Management.md` | Project_Architecture (v1.0), CICD_Pipeline_Deployment (v1.0) | 1.0 | Active |

### Governance

| Spec File | Depends On | Version | Status |
|---|---|---|---|
| `Dependency_Manifest.md` | None (root document) | 1.0 | Active |
| `Spec_Versioning.md` | Dependency_Manifest (v1.0) | 1.0 | Active |

## Status Definitions
- **Active** — current, verified consistent with its dependencies.
- **Stale** — a dependency has changed version since this file was last reviewed. Needs re-check.
- **Draft** — file created but not yet reviewed/tested.
- **Deprecated** — no longer in use, kept for historical reference only.

## Implementation Rules
- Every new spec file must add a row to the Manifest Table as part of being marked "done."
- No spec file is considered complete until its row exists here with accurate `Depends On` entries.
- If a file has zero dependencies (e.g., a foundational Architecture doc), its `Depends On` column should explicitly say "None" rather than being left blank.

## Acceptance Criteria
- [ ] Every spec file in the repo has exactly one row in the Manifest Table.
- [ ] No file's `Depends On` column references a file that doesn't exist in the repo.
- [ ] No dependency violates the one-way direction rule (Architecture → Frameworks → Components → Standards).
- [ ] Version numbers in this table match the version header inside each respective file.

## AI Generation Notes
When an AI agent is asked to modify any spec file, it must:
1. Check this manifest for that file's current version and dependents.
2. Warn the user which dependent files may become stale as a result of the change, before making the edit.
3. Update this manifest's row for the modified file (bump version, update Last Modified) as part of completing the edit.
4. Never silently update a dependent file without flagging it to the user first — per the process defined in `Spec_Versioning.md`.

## Future Extension Points
- Could be converted to a machine-readable format (YAML/JSON) once the number of specs grows large enough that manual table maintenance becomes error-prone.
- Could be paired with a script/agent that automatically scans all `.md` files for a `Depends On:` header and cross-validates against this manifest, catching mismatches automatically.