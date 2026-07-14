# Spec Versioning

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** Dependency_Manifest.md (v1.0)
**Category:** Governance

## Purpose
Defines how spec files are versioned, how changes propagate to dependent files, and how exceptions (cases where a real table doesn't fit the standard specs) are tracked. This is the operational companion to `Dependency_Manifest.md` — the Manifest tracks *what depends on what*; this document defines *what you do when something changes*.

## Scope
Covers the spec lifecycle (version, change, review, propagate). Does NOT cover runtime config changes (changing a table's `load_type` in `cfg_source` is just a config update, not a spec change) — this document only governs changes to the `.md` files in `/specs/`.

---

## Spec Versioning Rules

### Version Header
Every spec file carries a version header block at the top:

```
**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** Config_Framework.md (v1.0), Silver_Framework.md (v1.0)
**Category:** Frameworks
```

### Version Numbering

| Change Type | Version Bump | Example |
|---|---|---|
| New field/section added, no existing behavior changed | Minor: `1.0 → 1.1` | Adding a new config field to `Config_Framework.md` |
| Existing behavior/field changed or removed | Major: `1.0 → 2.0` | Changing `dq_rules` from flat string to object (the amendment we did earlier) |
| Typo fix, wording clarification, no behavioral change | No bump | Fixing a spelling error |

### Pre-Commit Exception
While all files are uncommitted (our current state), version tracking is informational only — all files stay at v1.0 and edits are made directly. Formal versioning begins from the first Git commit onward.

---

## Change Management Process

### When a Spec File Changes

```
1. Identify the file to change.
2. Check Dependency_Manifest.md → find all files listing this one in their "Depends On."
3. Assess impact:
   - Does the change affect any field/behavior those dependent files reference?
   - If YES → those files need review/update.
   - If NO → those files just need their "Depends On" version number bumped.
4. Make the change.
5. Update Dependency_Manifest.md:
   - Bump the changed file's version and Last Modified date.
   - Mark affected dependents as "Stale" until reviewed.
6. Review/update each Stale dependent.
7. Flip Stale dependents back to "Active" once confirmed consistent.
```

### Change Impact Classification

| Impact Level | What Changed | Action Required on Dependents |
|---|---|---|
| None | Typo, wording clarification | No action — dependents are unaffected |
| Low | New optional field/section added | Dependents bump version reference, no content change |
| Medium | Existing field's type or structure changed | Dependents review their references to the changed field, update if needed |
| High | Field removed, behavior reversed, or contract broken | Dependents almost certainly need content changes — treat as mandatory review |

### Who Does This
During the spec-writing phase: the person building the specs (you + the AI assistant) handles this inline as we go — which is exactly what we've been doing (flagging downstream impacts, patching `Config_Framework.md` when `Data_Quality_Framework.md` revealed a gap, etc.).

After deployment, when specs evolve over time: whoever proposes a spec change is responsible for running the Dependency Manifest check before the change is merged. If using AI agents to modify specs, the agent must follow this same process (per the AI Generation Notes in `Dependency_Manifest.md`).

---

## Onboarding Exceptions Log

When a new table is onboarded and it doesn't fit the standard spec-driven patterns (e.g., a source table with a non-standard CDC mechanism, or a business rule that can't be expressed as a declarative DQ expression), the exception is logged here rather than silently forking the spec or hacking the generated code.

### Exception Log Template

| Date | Table | Spec Affected | Exception Description | Resolution | Spec Amendment Needed? |
|---|---|---|---|---|---|
| *(populated as exceptions arise — empty at framework launch)* | | | | | |

### Rules for Exceptions
- An exception is NOT permission to hardcode a workaround — it's a tracked deviation that should lead to either a spec amendment (if the exception is generalizable) or a documented one-off with a clear justification.
- If the same exception occurs for 3+ tables, that's a signal the spec itself needs updating, not that 3+ exceptions should be logged — escalate to a spec change via the Change Management Process above.
- Exceptions must be reviewed periodically (e.g., quarterly) to identify patterns worth formalizing into the framework.

---

## Acceptance Criteria
- [ ] Every spec file has a version header with Version, Last Modified, Depends On, and Category.
- [ ] `Dependency_Manifest.md` is updated every time a spec changes (version bump + dependent status).
- [ ] No spec change is made without checking its downstream impact first.
- [ ] Exceptions are logged with enough detail to determine whether a spec amendment is warranted.

## Dependencies
- `Dependency_Manifest.md` (v1.0) — the tracking mechanism this governance process operates on.

## Future Extension Points
- Could add automated impact analysis (a script that parses all `Depends On` headers and generates the dependency graph automatically, flagging stale files) once the spec count grows large enough to make manual Manifest maintenance error-prone.

## AI Generation Notes
Any AI agent modifying a spec file must follow the Change Management Process above — specifically steps 2-5 (check manifest, assess impact, make change, update manifest). An agent that modifies a spec without updating the Manifest is violating governance, even if the content change itself is correct.