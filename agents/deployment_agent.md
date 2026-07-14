# Deployment Agent

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** orchestrator_agent.md (common contract), Config_Framework.md, CICD_Pipeline_Deployment.md, Secrets_Management.md, Naming_Standards.md
**Category:** Agents

## Purpose
Generates the infrastructure artifacts needed before any pipeline runs — config table DDL, supporting table DDL (schema_registry, watermark_registry, log/audit tables), setup notebooks, and environment-specific initialization scripts.

## What This Agent Reads
| Source | What It Extracts |
|---|---|
| `Config_Framework.md` | All config table schemas (`cfg_source`, `cfg_pipeline`, `cfg_workflow`, `cfg_validation`, `reg_schema`, `reg_watermark`) |
| `Logging_Framework.md` | Log table schema |
| `Audit_Framework.md` | Audit table schema |
| `CICD_Pipeline_Deployment.md` | Environment separation (schema prefixes), environment parameter |
| `Secrets_Management.md` | Secret scope naming convention (for documentation/setup instructions, not actual secret creation) |
| `Naming_Standards.md` | Table naming patterns (`cfg_`, `reg_`, `log_`, `audit_`) |

## What This Agent Generates
| Artifact | Location | Naming |
|---|---|---|
| Config table DDL | `src/sql/ddl/create_config_tables.sql` | One combined DDL file |
| Supporting table DDL | `src/sql/ddl/create_supporting_tables.sql` | Schema registry, watermark registry, profile log |
| Log/audit table DDL | `src/sql/ddl/create_log_audit_tables.sql` | Log and audit tables |
| Setup notebook | `src/notebooks/setup/setup_framework.py` | Runs all DDL, creates Unknown member dimension rows |
| Secret scope setup guide | `src/notebooks/setup/setup_secrets_guide.md` | Human-readable instructions (not executable — scopes are created manually) |

## Generation Rules
- All DDL must use the `{environment}` schema prefix pattern — the setup notebook accepts `environment` and creates tables in the correct schema.
- DDL must be idempotent (`CREATE TABLE IF NOT EXISTS`) — safe to re-run without error.
- Setup notebook must create all required tables, then verify they exist before returning success.
- Secret scope guide documents what scopes and keys to create manually — never attempts to create them via code.

## Acceptance Criteria
- [ ] All config, supporting, log, and audit tables are covered by DDL.
- [ ] DDL is idempotent — re-running creates nothing new if tables already exist.
- [ ] Setup notebook accepts `environment` parameter and creates tables in the correct schema.
- [ ] Secret scope guide is documentation only, not executable automation.