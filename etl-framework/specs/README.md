# Specs

This folder contains the full specification set for the ETL framework, organized by concern:

- `Architecture/` — overall project and medallion architecture, repo layout
- `Frameworks/` — the functional frameworks (config, ingestion, raw/silver/gold, SCD2, logging, audit, DQ, error handling, schema mgmt, orchestration)
- `Components/` — individual reusable components referenced by the frameworks
- `Standards/` — coding, naming, notebook, and error-handling standards
- `Testing/` — testing strategy and acceptance criteria
- `Deployment/` — environment promotion, CI/CD, secrets, asset bundles, rollback
- `Governance/` — dependency manifest, spec versioning, change management
- `AI_Generation/` — rules and contracts for AI-assisted generation of code from these specs
