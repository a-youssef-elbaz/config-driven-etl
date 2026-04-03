# config-driven-etl

**The Single Source of Truth for Your Metadata-Driven ETL Framework**

This repository manages all configuration data for the `ds_cil_bq_code_base.SP_DYNAMIC_ROUTINE_GENERATOR` stored procedure in BigQuery. Instead of writing individual ETL procedures, we store ETL logic as metadata in configuration tables. This repo is where that metadata lives as code — versioned, reviewed, and deployed through pipelines.

## What Lives Here

- User stories (business requirements for each ETL rule)
- Low-level designs (technical translation of user stories)
- SQL configuration scripts (INSERT statements that populate config tables)
- SQL rollback scripts (DELETE statements to undo config changes)
- The stored procedure itself and its DDL
- AI prompt templates for generating LLDs and configs
- System documentation (design doc, flowchart, examples, guides)
- CI/CD pipeline definitions

## Quick Start

1. **Create a branch** with format `feature/PBMIS-###-short-description`
2. **Write a User Story** in `products/{product}/US-{product}-{process}.md`
3. **Generate LLD** using Copilot Chat with the template in `prompts/us-to-lld.md`
4. **Generate CONFIG + ROLLBACK** using Copilot Chat with the template in `prompts/lld-to-config.md`
5. **Push and open a PR** targeting the `dev` branch
6. **Validation pipeline** runs automatically to check naming, SQL syntax, and completeness
7. **Get approved** and merge to trigger automatic deployment to DEV

## Repository Structure

```
etl-metadata-config/
├── .github/
│   ├── copilot-instructions.md
│   └── workflows/
│       ├── validate-pr.yml
│       └── deploy-config.yml
├── docs/
│   ├── DESIGN_DOCUMENT_v6.md
│   ├── FLOWCHART_v6.txt
│   ├── CONFIG_FILLING_GUIDE.md
│   ├── Config_Examples.sql
│   └── PRODUCT_REGISTRY.md
├── procedure/
│   ├── ds_cil_bq_code_base.SP_DYNAMIC_ROUTINE_GENERATOR.sql
│   ├── sp_log_execution_start.sql
│   ├── sp_log_execution_success.sql
│   ├── sp_log_execution_failure.sql
│   └── Data_Model.sql
├── prompts/
│   ├── us-to-lld.md
│   └── lld-to-config.md
├── shared/
│   ├── expressions.sql
│   └── expressions.rollback.sql
├── products/
│   ├── kk/
│   ├── sp/
│   ├── npk/
│   └── ...
├── releases/
├── scripts/
└── README.md
```

## File Naming Convention

```
{TYPE}-{product}-{process}.{ext}
```

- **TYPE**: US | LLD | CONFIG | ROLLBACK
- **product**: product abbreviation (matches folder name)
- **process**: what the ETL does (lowercase, hyphens)
- **ext**: .md for documents, .sql for SQL

## The Four Files Always Travel Together

For every process, exactly four files must exist:

```
products/kk/
├── US-kk-base-load.md           ← 1. Input (human-written)
├── LLD-kk-base-load.md          ← 2. Design (AI-generated)
├── CONFIG-kk-base-load.sql      ← 3. Deploy artifact (AI-generated)
└── ROLLBACK-kk-base-load.sql    ← 4. Undo artifact (AI-generated)
```

The validation pipeline checks that all four exist and share the same suffix.

## Two Development Approaches

### Approach A: IDE Chat Workflow (Copilot in VS Code)
1. Create branch and write User Story
2. Open Copilot Chat in VS Code
3. Generate LLD using `@workspace` and reference `prompts/us-to-lld.md`
4. Save the LLD file
5. Generate CONFIG + ROLLBACK using `prompts/lld-to-config.md`
6. Save both files
7. Push and open PR

### Approach B: Copilot Workspace Workflow
1. Create branch and write User Story
2. Push to repo and open GitHub Issue or PR
3. Click "Open in Workspace" on GitHub
4. Describe the task referencing the prompt templates
5. Review proposed changes
6. Copilot creates the PR for you

Both approaches produce identical output and go through the same validation and deployment pipelines.

## Branch Strategy

**Permanent Environment Branches:**
- `dev` (Development)
- `sit` (System Integration Testing)
- `uat` (User Acceptance Testing)
- `preprod` (Pre-production)
- `main` (Production)

**Promotion Path:**
`feature/PBMIS-###` → `dev` → `sit` → `uat` → `preprod` → `main`

## Key Documentation

For more detailed information, see:

- **[REPOSITORY_GUIDE.md](./REPOSITORY_GUIDE.md)** — Complete guide covering all sections, workflows, pipelines, and implementation phases
- **[docs/DESIGN_DOCUMENT_v6.md](./docs/DESIGN_DOCUMENT_v6.md)** — Full system design reference
- **[docs/CONFIG_FILLING_GUIDE.md](./docs/CONFIG_FILLING_GUIDE.md)** — Table schemas and validation rules
- **[docs/Config_Examples.sql](./docs/Config_Examples.sql)** — Example configurations with generated SQL
- **[docs/PRODUCT_REGISTRY.md](./docs/PRODUCT_REGISTRY.md)** — Product abbreviations and descriptions
- **[.github/copilot-instructions.md](./.github/copilot-instructions.md)** — Persistent AI context for Copilot