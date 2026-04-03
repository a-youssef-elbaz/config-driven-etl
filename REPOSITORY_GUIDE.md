# ETL Metadata Configuration Repository — Complete Guide

## The Single Source of Truth for Your Metadata-Driven ETL Framework

---

## Table of Contents

1. [Purpose of This Repository](#1-purpose-of-this-repository)
2. [Repository Structure](#2-repository-structure)
3. [File Types and Naming Conventions](#3-file-types-and-naming-conventions)
4. [Developer Responsibilities vs Automation](#4-developer-responsibilities-vs-automation)
5. [Approach A: IDE Chat Workflow (Copilot in VS Code)](#5-approach-a-ide-chat-workflow)
6. [Approach B: Copilot Workspace Workflow](#6-approach-b-copilot-workspace-workflow)
7. [How Both Approaches Converge](#7-how-both-approaches-converge)
8. [The User Story File Format](#8-the-user-story-file-format)
9. [The Prompt Templates](#9-the-prompt-templates)
10. [The Shared Expressions File](#10-the-shared-expressions-file)
11. [The Product Registry](#11-the-product-registry)
12. [The Copilot Instructions File](#12-the-copilot-instructions-file)
13. [The Validation Pipeline](#13-the-validation-pipeline)
14. [The Deployment Pipeline](#14-the-deployment-pipeline)
15. [The Rollback Strategy](#15-the-rollback-strategy)
16. [The Release Management Process](#16-the-release-management-process)
17. [Branch Strategy and PR Rules](#17-branch-strategy-and-pr-rules)
18. [Environment Strategy](#18-environment-strategy)
19. [Day-by-Day Walkthrough: New Rule Lifecycle](#19-day-by-day-walkthrough)
20. [Implementation Phases](#20-implementation-phases)
21. [Quick Reference Card](#21-quick-reference-card)

---

## 1. Purpose of This Repository

This repository manages **all configuration data** for the `ds_cil_bq_code_base.SP_DYNAMIC_ROUTINE_GENERATOR` stored procedure in BigQuery. Instead of writing individual ETL procedures, we store ETL logic as metadata in configuration tables. This repo is where that metadata lives as code — versioned, reviewed, and deployed through pipelines.

**What lives here:**
- User stories (business requirements for each ETL rule)
- Low-level designs (technical translation of user stories)
- SQL configuration scripts (INSERT statements that populate config tables)
- SQL rollback scripts (DELETE statements to undo config changes)
- The stored procedure itself and its DDL
- AI prompt templates for generating LLDs and configs
- System documentation (design doc, flowchart, examples, guides)
- CI/CD pipeline definitions

**What does NOT live here:**
- Business data
- Source table DDLs (those belong to the data platform repo)
- Airflow DAGs (those belong to the orchestration repo)
- Application code

---

## 2. Repository Structure

```
etl-metadata-config/
│
├── .github/
│   ├── copilot-instructions.md          ← Persistent AI context (see Section 12)
│   └── workflows/
│       ├── validate-pr.yml              ← Runs on every PR: checks naming,
│       │                                   syntax, completeness
│       └── deploy-config.yml            ← Runs on merge to main: deploys
│                                           config to DEV -> SIT -> UAT -> PREPROD -> PROD
│
├── docs/                                 ← System-level documentation
│   ├── DESIGN_DOCUMENT_v6.md            ← Full system design reference
│   ├── FLOWCHART_v6.txt                 ← Technical flowchart (all 14 sections)
│   ├── CONFIG_FILLING_GUIDE.md          ← Table schemas, decision trees,
│   │                                       validation rules
│   ├── Config_Examples.sql                  ← All 11 example configs with
│   │                                       generated SQL shown
│   ├── PROMPT_FOR_NEW_CHAT.md           ← Context prompt for new AI sessions
│   └── PRODUCT_REGISTRY.md             ← Maps product abbreviations to
│                                           full names (see Section 11)
│
├── procedure/                            ← The engine (rarely changes)
│   ├── ds_cil_bq_code_base.SP_DYNAMIC_ROUTINE_GENERATOR.sql ← Main stored procedure v6.2/v6.3
│   ├── sp_log_execution_start.sql       ← Logging sub-procedure
│   ├── sp_log_execution_success.sql     ← Logging sub-procedure
│   ├── sp_log_execution_failure.sql     ← Logging sub-procedure
│   └── Data_Model.sql                   ← DDL for all 10 config + audit tables
│
├── prompts/                              ← AI prompt templates
│   ├── us-to-lld.md                     ← Prompt 1: User Story → LLD
│   └── lld-to-config.md                 ← Prompt 2: LLD → CONFIG + ROLLBACK
│
├── shared/                               ← Cross-product shared configurations
│   ├── expressions.sql                  ← ALL reusable expressions (one file)
│   └── expressions.rollback.sql         ← Conditional rollback for expressions
│
├── products/                             ← ONE folder per product
│   ├── kk/                              ← Kontokorrent (current accounts)
│   │   ├── US-kk-base-load.md
│   │   ├── LLD-kk-base-load.md
│   │   ├── CONFIG-kk-base-load.sql
│   │   └── ROLLBACK-kk-base-load.sql
│   │
│   ├── sp/                              ← Sparprodukte (savings products)
│   │   ├── US-sp-daily-merge.md
│   │   ├── LLD-sp-daily-merge.md
│   │   ├── CONFIG-sp-daily-merge.sql
│   │   └── ROLLBACK-sp-daily-merge.sql
│   │
│   ├── npk/                             ← Non-Performing Kredit
│   │   ├── US-npk-classification.md
│   │   ├── LLD-npk-classification.md
│   │   ├── CONFIG-npk-classification.sql
│   │   └── ROLLBACK-npk-classification.sql
│   │
│   └── ... (more products)
│
├── releases/                             ← Combined deploy/rollback per release
│   ├── RELEASE-2024-01-15.sql           ← All configs for this release
│   ├── RELEASE-2024-01-15.rollback.sql  ← Undo entire release
│   └── ...
│
├── scripts/                              ← Automation utilities
│   ├── validate_config.py               ← Checks naming, SQL syntax, CONFIG
│   │                                       vs ROLLBACK completeness
│   ├── build_release.py                 ← Combines product configs into
│   │                                       release files
│   └── deploy.py                        ← Executes SQL against BigQuery
│
└── README.md                             ← Getting started, links to docs
```

### Design Principles Behind This Structure

**Principle 1: Everything about a product lives in one folder.**
When you look for "KK base load," you open `products/kk/` and find the user story, the LLD, the config, and the rollback — all in one place. No hunting across multiple directories.

**Principle 2: Flat until complexity demands depth.**
Product folders are flat (no subfolders). If a product grows to have 15+ rules and the folder feels crowded, subfolders can be added at that point. Don't pre-optimize.

**Principle 3: Shared resources are separated.**
Reusable expressions don't belong to any single product, so they live in `shared/`. One file for all expressions keeps them easy to browse and manage.

**Principle 4: Infrastructure files are stable.**
`procedure/`, `docs/`, `prompts/`, and `.github/` change rarely. The action happens in `products/` and `releases/`. This separation means developers focus on product folders and don't accidentally touch system files.

---

## 3. File Types and Naming Conventions

### File Naming Pattern

```
{TYPE}-{product}-{process}.{ext}

Where:
  {TYPE}     = US | LLD | CONFIG | ROLLBACK
  {product}  = product abbreviation (matches folder name)
  {process}  = what the ETL does (lowercase, hyphens)
  {ext}      = .md for documents, .sql for SQL
```

### File Types

```
┌──────────┬───────────────┬────────────────────────────────────────────────┐
│ Type     │ Extension     │ What It Contains                               │
├──────────┼───────────────┼────────────────────────────────────────────────┤
│ US-      │ .md           │ User story: business requirements, metadata    │
│          │               │ header with rule_id, source tables, target     │
│          │               │ table, merge type. Written by the developer    │
│          │               │ or business analyst.                           │
├──────────┼───────────────┼────────────────────────────────────────────────┤
│ LLD-     │ .md           │ Low-level design: technical translation of     │
│          │               │ the US. Source-to-target mapping, join logic,  │
│          │               │ transform rules, column definitions.           │
│          │               │ Generated by Copilot.                          │
├──────────┼───────────────┼────────────────────────────────────────────────┤
│ CONFIG-  │ .sql          │ SQL INSERT statements that populate all        │
│          │               │ config tables for one rule. Ready to execute   │
│          │               │ against BigQuery. Generated by Copilot.        │
├──────────┼───────────────┼────────────────────────────────────────────────┤
│ ROLLBACK-│ .sql          │ SQL DELETE statements that undo the CONFIG     │
│          │               │ file. One DELETE per table, keyed on rule_id.  │
│          │               │ Reverse dependency order. Generated by         │
│          │               │ Copilot alongside the CONFIG.                  │
└──────────┴───────────────┴────────────────────────────────────────────────┘
```

### Folder Naming Rules

```text
Product folders: alphanumeric, hyphens allowed, abbreviations OK
  Good:  KK, NPK, cust-360, mis
  Bad:   Product_A, riskMgmt, my product

Validation regex: ^[a-zA-Z][a-zA-Z0-9-]*$
```

### The Four Files Always Travel Together

For every process, exactly four files must exist:

```
products/kk/
├── US-kk-base-load.md           ← 1. Input (human-written)
├── LLD-kk-base-load.md          ← 2. Design (AI-generated)
├── CONFIG-kk-base-load.sql      ← 3. Deploy artifact (AI-generated)
└── ROLLBACK-kk-base-load.sql    ← 4. Undo artifact (AI-generated)
```

The validation pipeline checks that all four exist and share the same suffix. If any file is missing, the PR fails.

### Products With Multiple Rules

A product may have multiple ETL rules. Each rule gets its own set of four files:

```
products/kk/
├── US-kk-base-load.md
├── LLD-kk-base-load.md
├── CONFIG-kk-base-load.sql
├── ROLLBACK-kk-base-load.sql
├── US-kk-monthly-agg.md           ← Second rule for same product
├── LLD-kk-monthly-agg.md
├── CONFIG-kk-monthly-agg.sql
└── ROLLBACK-kk-monthly-agg.sql
```

---

## 4. Developer Responsibilities vs Automation

### The Golden Rule: Humans Create Structure, Automation Creates Content

```
┌──────────────────────────────────┬──────────────┬───────────────────────────┐
│ Action                           │ Done By      │ When                      │
├──────────────────────────────────┼──────────────┼───────────────────────────┤
│ Create product folder            │ Developer    │ When a new product starts │
│ (e.g., products/kk/)             │              │                           │
├──────────────────────────────────┼──────────────┼───────────────────────────┤
│ Write user story file            │ Developer /  │ For each new ETL rule     │
│ (US-*.md)                        │ Analyst      │                           │
├──────────────────────────────────┼──────────────┼───────────────────────────┤
│ Generate LLD file                │ Copilot      │ After US is written       │
│ (LLD-*.md)                       │ (via Chat    │ (Approach A: developer    │
│                                  │  or          │  saves the output.        │
│                                  │  Workspace)  │  Approach B: Copilot      │
│                                  │              │  commits directly.)       │
├──────────────────────────────────┼──────────────┼───────────────────────────┤
│ Generate CONFIG + ROLLBACK       │ Copilot      │ After LLD is generated    │
│ (CONFIG-*.sql + ROLLBACK-*.sql)  │ (both files  │ (same prompt generates    │
│                                  │  together)   │  both files as a pair)    │
├──────────────────────────────────┼──────────────┼───────────────────────────┤
│ Update PRODUCT_REGISTRY.md       │ Copilot      │ When creating new product │
│                                  │              │ folder (add one row)      │
├──────────────────────────────────┼──────────────┼───────────────────────────┤
│ Update shared/expressions.sql    │ Copilot      │ When a new reusable       │
│                                  │              │ expression is generated   │
├──────────────────────────────────┼──────────────┼───────────────────────────┤
│ Validate PR                      │ Pipeline     │ Automatic on every PR     │
│                                  │ (automatic)  │                           │
├──────────────────────────────────┼──────────────┼───────────────────────────┤
│ Review and approve PR            │ You / Senior │ Before merge              │
│                                  │ architect    │                           │
├──────────────────────────────────┼──────────────┼───────────────────────────┤
│ Deploy to DEV                    │ Pipeline     │ Automatic on merge to dev │
│                                  │ (automatic)  │                           │
├──────────────────────────────────┼──────────────┼───────────────────────────┤
│ Promote & Deploy                 │ Pipeline     │ Triggers via PR merges    │
│ (Develop, SIT, UAT, PREPROD, PROD) │ (automatic)  │ to target env branches    │
├──────────────────────────────────┼──────────────┼───────────────────────────┤
│ Build release file               │ Script       │ When cutting a release    │
│                                  │ (triggered   │ (combines individual      │
│                                  │  by you)     │  configs into one file)   │
└──────────────────────────────────┴──────────────┴───────────────────────────┘
```

### What the Developer Never Has to Do

- Write deployment scripts or pipelines (done once during setup).
- Manually execute SQL against BigQuery (pipeline does it).
- Write rollback scripts separately (Copilot generates them with the CONFIG).
- Remember the table dependency order (the prompt template handles it).
- Worry about naming validation (pipeline catches violations).

---

## 5. Approach A: IDE Chat Workflow (Copilot in VS Code)

This is the primary workflow. The developer uses Copilot Chat inside VS Code to generate files, saves them manually, and pushes to the repo.

### Step-by-Step

```
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 1: Create Branch                                               │
│                                                                     │
│ Developer creates a feature branch:                                 │
│   git checkout -b feature/kk-base-load                              │
│                                                                     │
│ If this is a NEW product (folder doesn't exist):                    │
│   mkdir products/kk                                                 │
│   Ask Copilot to update docs/PRODUCT_REGISTRY.md                    │
└──────────────────────────────────┬──────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 2: Write User Story                                            │
│                                                                     │
│ Developer creates: products/kk/US-kk-base-load.md                   │
│ (See Section 8 for the required format with metadata header)        │
│                                                                     │
│ The US contains: rule_id, source tables, target table,              │
│ merge type, business logic description.                             │
└──────────────────────────────────┬──────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 3: Generate LLD via Copilot Chat                               │
│                                                                     │
│ Developer opens Copilot Chat in VS Code and types:                  │
│                                                                     │
│   @workspace Using the template in #file:prompts/us-to-lld.md,      │
│   generate an LLD for #file:products/kk/US-kk-base-load.md.         │
│   Reference #file:docs/CONFIG_FILLING_GUIDE.md for table schemas.   │
│                                                                     │
│ Copilot generates the LLD document.                                 │
│ Developer reviews it in the chat.                                   │
│ Developer saves it as: products/kk/LLD-kk-base-load.md              │
└──────────────────────────────────┬──────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 4: Generate CONFIG + ROLLBACK via Copilot Chat                 │
│                                                                     │
│ Developer continues in Copilot Chat:                                │
│                                                                     │
│   Using the template in #file:prompts/lld-to-config.md,             │
│   generate CONFIG and ROLLBACK SQL for                              │
│   #file:products/kk/LLD-kk-base-load.md.                            │
│   Reference #file:docs/Config_Examples.sql for correct patterns.        │
│                                                                     │
│ Copilot generates BOTH files in one response:                       │
│   - CONFIG: INSERT statements for all config tables                 │
│   - ROLLBACK: DELETE statements in reverse dependency order         │
│                                                                     │
│ Developer saves:                                                    │
│   products/kk/CONFIG-kk-base-load.sql                               │
│   products/kk/ROLLBACK-kk-base-load.sql                             │
└──────────────────────────────────┬──────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 5: Push and Open PR                                            │
│                                                                     │
│   git add products/kk/                                              │
│   git commit -m "feat(kk): add KK base load config"                 │
│   git push origin feature/kk-base-load                              │
│   → Open Pull Request on GitHub                                     │
│                                                                     │
│ From here, everything is automatic (see Sections 13-14).            │
└─────────────────────────────────────────────────────────────────────┘
```

### What the Developer Sees in Their Folder After Step 4

```
products/kk/
├── US-kk-base-load.md           ← They wrote this
├── LLD-kk-base-load.md          ← Copilot generated, they saved it
├── CONFIG-kk-base-load.sql      ← Copilot generated, they saved it
└── ROLLBACK-kk-base-load.sql    ← Copilot generated, they saved it
```

Four files. One product folder. Ready for PR.

---

## 6. Approach B: Copilot Workspace Workflow

This workflow uses GitHub Copilot Workspace (if available). Instead of chatting in VS Code and saving files manually, Copilot proposes file changes directly on GitHub and creates the PR.

### Step-by-Step

```
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 1: Create the Product Folder and User Story                    │
│                                                                     │
│ This step is the same as Approach A:                                │
│   - Create folder products/kk/ (if new product)                     │
│   - Write US-kk-base-load.md                                        │
│   - Push to a branch and open a PR (or create a GitHub Issue)       │
│                                                                     │
│ The US file must exist in the repo before Workspace can see it.     │
└──────────────────────────────────┬──────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 2: Open Copilot Workspace                                      │
│                                                                     │
│ On github.com, navigate to the PR or issue.                         │
│ Click "Open in Workspace" (if available).                           │
│                                                                     │
│ Describe the task:                                                  │
│   "Generate LLD, CONFIG, and ROLLBACK files for the user story      │
│    in products/kk/US-kk-base-load.md. Follow the templates in       │
│    prompts/us-to-lld.md and prompts/lld-to-config.md. Use the       │
│    examples in docs/Config_Examples.sql for correct SQL patterns."      │ 
│                                                                     │
│ Copilot Workspace reads the repo files for context.                 │
└──────────────────────────────────┬──────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 3: Review Proposed Changes                                     │
│                                                                     │
│ Copilot Workspace shows you a plan:                                 │
│   "I will create 3 files:                                           │
│    - products/kk/LLD-kk-base-load.md                                │
│    - products/kk/CONFIG-kk-base-load.sql                            │
│    - products/kk/ROLLBACK-kk-base-load.sql"                         │
│                                                                     │
│ It shows the CONTENT of each file for you to review.                │
│ You can edit any file directly in the Workspace UI.                 │
│ You can ask Copilot to regenerate specific parts.                   │
└──────────────────────────────────┬──────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 4: Create PR                                                   │
│                                                                     │
│ Click "Create Pull Request."                                        │
│ Copilot commits the files to a new branch and opens the PR.         │
│ No manual file saving, no git commands.                             │
│                                                                     │
│ From here, the same validation and deployment pipelines run.        │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Differences From Approach A

```
┌─────────────────────────┬────────────────────────┬──────────────────────┐
│ Aspect                  │ Approach A (IDE Chat)  │Approach B (Workspace)│
├─────────────────────────┼────────────────────────┼──────────────────────┤
│ Where you work          │ VS Code on your        │ github.com in        │
│                         │ machine                │ browser              │
├─────────────────────────┼────────────────────────┼──────────────────────┤
│ How you talk to Copilot │ Chat panel in IDE      │ Task description in  │
│                         │                        │ Workspace UI         │
├─────────────────────────┼────────────────────────┼──────────────────────┤
│ How files are saved     │ You copy output and    │ Copilot creates files│
│                         │ save manually          │ directly in the repo │
├─────────────────────────┼────────────────────────┼──────────────────────┤
│ How PR is created       │ You run git commands   │ Copilot creates the  │
│                         │ and open PR manually   │ PR for you           │
├─────────────────────────┼────────────────────────┼──────────────────────┤
│ Iterating on output     │ Continue chatting,     │ Edit in Workspace UI │
│                         │ update files locally   │ or ask Copilot to    │
│                         │                        │ regenerate           │
├─────────────────────────┼────────────────────────┼──────────────────────┤
│ Availability            │ Available now          │ May not be available │
│                         │                        │ in org yet           │
├─────────────────────────┼────────────────────────┼──────────────────────┤
│ Output files            │ Identical              │ Identical            │
├─────────────────────────┼────────────────────────┼──────────────────────┤
│ What happens after PR   │ Identical              │ Identical            │
└─────────────────────────┴────────────────────────┴──────────────────────┘
```

---

## 7. How Both Approaches Converge

Regardless of which approach creates the files, everything after the PR is identical:

```
      Approach A                    Approach B
   (IDE Chat + manual)         (Copilot Workspace)
          │                            │
          │   4 files in products/     │
          │   folder + PR opened       │
          └──────────┬─────────────────┘
                     │
                     ▼
        ┌────────────────────────┐
        │  VALIDATION PIPELINE   │  ← Runs automatically on PR
        │                        │
        │  • File naming check   │
        │  • SQL syntax check    │
        │  • CONFIG ↔ ROLLBACK   │
        │    completeness check  │
        │  • Rule ID uniqueness  │
        │  • Product folder      │
        │    matches file prefix │
        └───────────┬────────────┘
                    │
                    ▼
        ┌────────────────────────┐
        │  HUMAN REVIEW          │  ← You approve the PR
        └───────────┬────────────┘
                    │
                    ▼
        ┌────────────────────────┐
        │  DEPLOYMENT PIPELINE   │  ← Runs on PR merge
        │                        │
        │  • PR to dev -> DEV    │
        │  • PR to sit -> SIT    │
        │  • PR to uat -> UAT    │
        │  • PR to preprod       │
        │  • PR to main -> PROD  │
        └────────────────────────┘
```

**The pipeline does not know or care which approach was used.** It only sees: "There are 4 correctly named files in a product folder. The SQL is valid. The ROLLBACK matches the CONFIG. Proceed."

This means your team can use both approaches simultaneously. Developer Ahmed prefers VS Code Chat. Developer Fatima prefers Copilot Workspace. Both produce the same output, go through the same pipeline, and deploy the same way.

---

## 8. The User Story File Format

The User Story should be written in natural language, storing info about source_id, rule_id, source_tables, and target_table. Copilot easily extracts these details from the text.

### Template

```markdown
# User Story: {Short Title}

## Business Requirement
{Free-form description of what the ETL should do.
 Tell the story naturally but ensure you reference the Rule ID, Source ID, Target Dataset, and
 Target Table so the AI can pick them up.
 Example: "We need a new rule (Rule ID: KK_BASE_LOAD) coming from source MIS_KK..."}

## Source Tables

{List the source tables involved, what each contains,
 and any filtering conditions.}

## Business Rules / Transformations

{Describe any calculations, derivations, filters,
 or enrichment logic.}

## Acceptance Criteria

{How to verify the ETL is working correctly.
 Expected row counts, spot-check queries, etc.}
```

### Example

```markdown
# User Story: KK Base Daily Load

## Business Requirement
As a MIS analyst, I need a rule (Rule ID: KK_BASE_LOAD) that takes source MIS_KK 
and uses an OVERWRITE strategy into target ds_stg.base_temp.

As a MIS analyst, I need the KK (Kontokorrent) base table
loaded daily with current account data enriched with lookup
attributes and previous-period aggregation context.

The load should overwrite only the current reference date
partition, preserving historical data.

## Source Tables

1. ds_stg.main_table — primary current account records
2. ds_stg.lookup_table — attribute lookup (status, effective date)
3. ds_inbound.agg_table — previous period aggregation
   (filtered to @prev_ref_date, grouped by grp_key)

## Business Rules / Transformations

1. Join main_table to lookup_table on key1 (LEFT JOIN).
   Exclude technical columns _PARTITIONTIME and _load_ts.
2. Parse raw_date field from YYYYMMDD string to DATE.
3. Join to previous-period aggregation for first_seen derivation.
4. UNNEST context fields (ctx_a, ctx_b) for context resolution.
5. Filter: only keep rows where ctx_resolved IN (1,2).
6. Derive tier: IF ctx_resolved = 1 THEN amt_a ELSE amt_b.

## Acceptance Criteria

- Row count in base_temp for @ref_date matches source after filter.
- No rows exist in base_temp with ctx_resolved outside (1,2).
- first_seen column populated for all rows with prior period data.
```

### Why We Kept it Free-Form

The validation pipeline no longer parses a rigid header. Instead, Copilot understands the standard text format to safely extract the metadata. This removes friction for business analysts writing the user stories while preserving technical accuracy.

---

## 9. The Prompt Templates

Two prompt files live in `prompts/`. These are the "instructions" that tell Copilot how to transform a US into an LLD, and an LLD into CONFIG + ROLLBACK.

### prompts/us-to-lld.md

This prompt tells Copilot how to read a user story and produce a low-level technical design. It should reference:
- The table schemas from CONFIG_FILLING_GUIDE.md.
- The CTE pipeline architecture from DESIGN_DOCUMENT_v6.md.
- The naming conventions from this guide.

The LLD output should contain:
- Source-to-target column mapping (every target column traced to its origin).
- Join specifications (type, condition, column selection per side).
- Transform specifications (pass-through mode, computed columns, filters).
- Output mapping (final SELECT with aliases).
- Merge key identification (which columns are keys vs values).

### prompts/lld-to-config.md

This prompt tells Copilot how to read an LLD and produce two outputs:

**Output 1: CONFIG file (INSERT statements)**
- Follow the Config Filling Order (9 steps from the flowchart).
- Reference Config_Examples.sql for correct syntax patterns.
- Include file header comment with product, process, rule_id, and generation date.
- Include the generated SQL as a comment block at the end (for review).
- **IMPORTANT: If a new reusable output expression is identified while creating the configs from LLD (and not existing/considered), you MUST create the INSERT statement for it directly in the generation step so it can be appended to shared/expressions.sql.**

**Output 2: ROLLBACK file (DELETE statements)**
- One DELETE per table that received an INSERT.
- Use `DELETE FROM ds_metadata.{table} WHERE rule_id = '{RULE_ID}'`.
- Order: reverse of CONFIG dependency order (children before parents).
- Include file header comment matching the CONFIG.

Both outputs are generated in a single Copilot response. The developer (Approach A) or Copilot Workspace (Approach B) separates them into two files.

---

## 10. The Shared Expressions File

### shared/expressions.sql

One file containing ALL reusable expressions across all products. Expressions are rows in the `t_pbmis_output_expression` table that can be referenced by `expression_id` from `t_pbmis_transform_column` or `t_pbmis_output_rule`.

**Structure of the file:**

```sql
-- ============================================================================
-- SHARED EXPRESSIONS — Reusable SQL fragments for t_pbmis_output_expression table
--
-- HOW TO ADD: Append new INSERT at the bottom of the relevant section.
-- HOW TO UPDATE: Add a new version row (do NOT modify existing rows).
-- HOW TO DEACTIVATE: Add UPDATE statement at the bottom of the file.
-- ============================================================================

-- ----------------------------------------------------------------------------
-- EXP_REF_DATE: Date casting of input param
-- ----------------------------------------------------------------------------
INSERT INTO ds_metadata_config.t_pbmis_output_expression
  (expression_id, expression_version, expression_description,
   expression_logic, is_active, change_notes)
VALUES
('EXP_REF_DATE', 1, 'Cast input parameter to YYYY-MM-DD',
 "CAST(@ref_date AS DATE FORMAT 'YYYY-MM-DD')",
 TRUE, 'Initial version');

-- ----------------------------------------------------------------------------
-- EXP_LOAD_USER: Get current session user
-- ----------------------------------------------------------------------------
INSERT INTO ds_metadata_config.t_pbmis_output_expression
  (expression_id, expression_version, expression_description,
   expression_logic, is_active, change_notes)
VALUES
('EXP_LOAD_USER', 1, 'Current BQ session user',
 "SESSION_USER()",
 TRUE, 'Initial version');

-- ----------------------------------------------------------------------------
-- EXP_COMPLEX_CALC: Complex calculation logic
-- ----------------------------------------------------------------------------
INSERT INTO ds_metadata_config.t_pbmis_output_expression
  (expression_id, expression_version, expression_description,
   expression_logic, is_active, change_notes)
VALUES
('EXP_COMPLEX_CALC', 1, 'Risk tier complex case when',
 "CASE WHEN amount > 1000000 AND status = 'A' THEN (amount * 0.05) WHEN amount > 10000 THEN (amount * 0.02) ELSE 0 END",
 TRUE, 'Initial version');

```

### shared/expressions.rollback.sql

Conditional DELETEs that only remove expressions if no other rule uses them:

```sql
-- ============================================================================
-- ROLLBACK: Shared Expressions
-- Only deletes expressions that are not referenced by any active rule.
-- ============================================================================

DELETE FROM ds_metadata_config.t_pbmis_output_expression
WHERE expression_id = 'EXP_REF_DATE'
  AND NOT EXISTS (
    SELECT 1 FROM ds_metadata_config.t_pbmis_transform_column
    WHERE expression_id = 'EXP_REF_DATE' AND is_active = TRUE
  )
  AND NOT EXISTS (
    SELECT 1 FROM ds_metadata_config.t_pbmis_output_rule
    WHERE expression_id = 'EXP_REF_DATE' AND is_active = TRUE
  );
```

### When a Product Needs a New Expression

If the LLD calls for a new reusable expression that doesn't exist in `shared/expressions.sql`, Copilot automatically generates the INSERT statement and appends it to the shared file in the same PR that adds the product config. The PR then contains:
- Changes to `shared/expressions.sql` (new rows appended).
- New files in `products/kk/` (the four product files).

The reviewer sees everything in one PR.

---

## 11. The Product Registry

### docs/PRODUCT_REGISTRY.md

A simple lookup table that maps abbreviations to full names. Automatically updated by Copilot whenever a new product folder is created.

```markdown
# Product Registry

| Folder | Full Name                  | Owner      | Description                        |
|--------|---------------------------|------------|------------------------------------|
| kk     | Kontokorrent              | MIS Team   | Current account base loading       |
| sp     | Sparprodukte              | MIS Team   | Savings products processing        |
| npk    | Non-Performing Kredit     | Risk Team  | NPL/NPK risk classification       |
| cust   | Customer 360              | CRM Team   | Customer dimension enrichment      |
```

This file exists for humans — the pipeline doesn't read it. Its purpose is to help newcomers and reviewers understand what "kk" or "npk" means.

---

## 12. The Copilot Instructions File

### .github/copilot-instructions.md

GitHub Copilot reads this file automatically and includes its content as context in every chat session within this repository. This is your "permanent system prompt."

```markdown
# Copilot Instructions for etl-metadata-config

## What This Repository Is
This repository manages metadata configuration for the
ds_cil_bq_code_base.SP_DYNAMIC_ROUTINE_GENERATOR stored procedure in BigQuery.
ETL logic is stored as data in config tables, not as code.

## Key Reference Files
- System constraints, architecture, schemas, and flowchart: docs/System_Technical_Reference.md
- Example configs with generated SQL: docs/Config_Examples.sql

## When Generating LLD Documents
Follow the template in prompts/us-to-lld.md.
Always include: source-to-target mapping, join specs,
transform specs, merge key identification.

## When Generating CONFIG + ROLLBACK SQL
Follow the template in prompts/lld-to-config.md.
Always generate BOTH the CONFIG and ROLLBACK files together.
Follow the 9-step Config Filling Order from the flowchart.
IMPORTANT: If the LLD identifies a NEW reusable expression, you MUST ALSO generate:
1. The INSERT statement to be appended to shared/expressions.sql
2. The conditional DELETE statement to be appended to shared/expressions.rollback.sql
Never place expression statements inside the product-specific sql files.

## File Naming Convention
- US-{product}-{process}.md
- LLD-{product}-{process}.md
- CONFIG-{product}-{process}.sql
- ROLLBACK-{product}-{process}.sql

## Rule ID Convention
Rule IDs come from the user story.
They are UPPER_SNAKE_CASE (e.g., KK_MONTHLY_CONSOLIDATION, NPK_DAILY_INBOUND).
Never invent a rule_id — always use the one from the US file.

## ROLLBACK File Rules
- One DELETE per config table that received an INSERT.
- Use: DELETE FROM ds_metadata_config.{table} WHERE rule_id = '{RULE_ID}'
- Order: reverse of CONFIG (children first, parents last):
  t_pbmis_merge_config → t_pbmis_output_rule → t_pbmis_output_config →
  t_pbmis_transform_column → t_pbmis_transform_config → t_pbmis_join_config →
  t_pbmis_source_config → t_pbmis_rule_config
- Do NOT include expression rollbacks in product ROLLBACK files.
  Expressions are managed in shared/expressions.rollback.sql.

## Constraints to Enforce
1. Never set both _select AND _exclude for the same join side.
2. Never set both join_to_alias AND join_to_inline_sql.
3. pass_through_except only valid when pass_through = 'ALL'.
4. Exactly ONE of expression_id OR source_column (never both, never neither).
5. t_pbmis_output_rule.source_column must NOT have table alias prefix.
6. Same constraint #4 applies to t_pbmis_output_rule.
```

This means every time a developer opens Copilot Chat in this repo, Copilot already knows the naming conventions, the file structure, the constraints, and where to find reference files. The developer doesn't need to explain the system from scratch each time.

---

## 13. The Validation Pipeline

### .github/workflows/validate-pr.yml

Runs automatically on every Pull Request. Checks everything before a human reviewer even looks at it.

### What It Checks

```
CHECK 1: File Naming Convention
─────────────────────────────────────────────────────────
For every file in products/:
  • Starts with US-, LLD-, CONFIG-, or ROLLBACK-
  • Product prefix in filename matches folder name
    (file in products/kk/ must start with {TYPE}-kk-)
  • Extension matches type (.md for US/LLD, .sql for CONFIG/ROLLBACK)

CHECK 2: File Completeness
─────────────────────────────────────────────────────────
For every process found (identified by the US- file):
  • LLD file exists
  • CONFIG file exists
  • ROLLBACK file exists
  All four files share the same {product}-{process} suffix.

CHECK 3: SQL Syntax
─────────────────────────────────────────────────────────
For every CONFIG-*.sql file:
  • All statements are valid INSERT INTO ds_metadata_config.{table}
  • Tables referenced are in the known list:
    t_pbmis_rule_config, t_pbmis_source_config, t_pbmis_join_config, t_pbmis_transform_config,
    t_pbmis_transform_column, t_pbmis_output_config, t_pbmis_output_rule, t_pbmis_merge_config,
    t_pbmis_output_expression
  • No DROP, TRUNCATE, UPDATE, or CREATE statements
    (CONFIG files only contain INSERTs/MERGEs)

CHECK 4: CONFIG ↔ ROLLBACK Completeness
─────────────────────────────────────────────────────────
For every CONFIG-*.sql / ROLLBACK-*.sql pair:
  • Every table in CONFIG has a matching DELETE in ROLLBACK
  • The rule_id in ROLLBACK WHERE clauses matches CONFIG
  • ROLLBACK does not contain INSERTs
  • ROLLBACK does not target tables outside ds_metadata_config

CHECK 5: Rule ID Uniqueness
─────────────────────────────────────────────────────────
  • Extract rule_id from each CONFIG file
  • Verify no two CONFIG files across ALL products
    share the same rule_id
  • (The validation pipeline validates uniqueness across
    all CONFIG SQL files, but no longer enforces matching
    a strict US file header structure)

CHECK 6: Product Folder Validation
─────────────────────────────────────────────────────────
  • Folder name matches regex: ^[a-zA-Z][a-zA-Z0-9-]*$
  • Folder exists in PRODUCT_REGISTRY.md (warning, not failure)

CHECK 7: Shared Expression Validation
─────────────────────────────────────────────────────────
For shared/expressions.sql and shared/expressions.rollback.sql:
  • Only contains INSERT/MERGE (for main) or DELETE (for rollback)
  • Targets ONLY ds_metadata_config.t_pbmis_output_expression
  • Rollback condition uses NOT EXISTS referencing active rules
```

### What Happens When a Check Fails

The PR gets a red status check with a clear message:

```
❌ Validation Failed

CHECK 2 FAILED: File completeness
  Missing: products/kk/ROLLBACK-kk-base-load.sql
  Expected: every CONFIG file must have a matching ROLLBACK file.
  
CHECK 5 FAILED: Rule ID uniqueness
  Rule ID 'KK_BASE_LOAD' found in:
    - products/kk/CONFIG-kk-base-load.sql (this PR)
    - products/kk/CONFIG-kk-base-load-v2.sql (existing)
  Each rule_id must be unique across the entire repository.
```

The developer fixes the issue and pushes again. The validation reruns automatically.

---

## 14. The Deployment Pipeline

### .github/workflows/deploy-env.yml

A single generic GitHub Action that runs on all environment branches. It uses GitHub Environments to inject the correct credentials and variables dynamically based on the target branch.

### Deployment Flow

```text
TRIGGER: PR merged to `dev`, `sit`, `uat`, `preprod`, or `main` (prod)
         AND changed files include products/*/CONFIG-*.sql
         OR shared/expressions.sql

STEP 1: RESOLVE ENVIRONMENT
  • If branch == 'dev' → Target Env: DEV, project: {projectId-dev}
  • If branch == 'sit' → Target Env: SIT, project: {projectId-sit}
  • If branch == 'uat' → Target Env: UAT, project: {projectId-uat}
  • If branch == 'preprod' → Target Env: PREPROD, project: {projectId-preprod}
  • If branch == 'main' → Target Env: PROD, project: {projectId-prod}

STEP 2: IDENTIFY CHANGES
  • List all CONFIG files that were added or modified
  • List if shared/expressions.sql was modified

STEP 3: ORDER STATEMENTS
  • Read all identified CONFIG files
  • Sort INSERT statements by table dependency:
    1. t_pbmis_output_expression
    2. t_pbmis_rule_config
    3. t_pbmis_source_config
    4. t_pbmis_join_config
    5. t_pbmis_transform_config
    6. t_pbmis_transform_column
    7. t_pbmis_output_config
    8. t_pbmis_output_rule
    9. t_pbmis_merge_config

STEP 4: DEPLOY TO TARGET BigQuery PROJECT
  • Authenticate via GCP Workload Identity in GitHub Actions
  • A Python script (e.g. `deploy_config.py`) reads the SQL files
  • Uses the `google-cloud-bigquery` library to run the statements
    as transactions against the target BigQuery environment
  • Note: `shared/expressions.sql` is always executed BEFORE
    the product CONFIG files so that the new expressions exist 
    when the rules are registered.

STEP 5: SMOKE TEST (DEV / SIT only)
  • For each new rule_id deployed, run smoke test
  • CALL ds_cil_bq_code_base.SP_DYNAMIC_ROUTINE_GENERATOR('{RULE_ID}', CURRENT_DATE(), 'CI_SMOKE_TEST')
  • Verify no errors raised

ON FAILURE AT ANY STEP:
  • Execute the ROLLBACK files for failed configs
  • Generate failure report natively on GitHub via Job Summaries 
    (writing markdown output directly to $GITHUB_STEP_SUMMARY)
  • Pipeline shows red status, halting the promotion path
```

---

## 15. The Rollback Strategy

### Three Levels of Rollback

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│ LEVEL 1: Single Rule Rollback                                       │
│ ─────────────────────────────                                       │
│                                                                     │
│ File: products/kk/ROLLBACK-kk-base-load.sql                         │
│ Scope: Undo one specific rule                                       │
│ When: A single rule has a bug in its config                         │
│ How: Execute the ROLLBACK file against BigQuery                     │
│                                                                     │
│ Example content:                                                    │
│   DELETE FROM ds_metadata_config.t_pbmis_merge_config               │
│     WHERE rule_id = 'KK_BASE_LOAD';                                 │
│   DELETE FROM ds_metadata_config.t_pbmis_output_rule                │
│     WHERE rule_id = 'KK_BASE_LOAD';                                 │
│   DELETE FROM ds_metadata_config.t_pbmis_output_config              │
│     WHERE rule_id = 'KK_BASE_LOAD';                                 │
│   DELETE FROM ds_metadata_config.t_pbmis_transform_column           │
│     WHERE rule_id = 'KK_BASE_LOAD';                                 │
│   DELETE FROM ds_metadata_config.t_pbmis_transform_config           │
│     WHERE rule_id = 'KK_BASE_LOAD';                                 │
│   DELETE FROM ds_metadata_config.t_pbmis_join_config                │
│     WHERE rule_id = 'KK_BASE_LOAD';                                 │
│   DELETE FROM ds_metadata_config.t_pbmis_source_config              │
│     WHERE rule_id = 'KK_BASE_LOAD';                                 │
│   DELETE FROM ds_metadata_config.t_pbmis_rule_config                │
│     WHERE rule_id = 'KK_BASE_LOAD';                                 │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ LEVEL 2: Release Rollback                                           │
│ ─────────────────────────                                           │
│                                                                     │
│ File: releases/RELEASE-2024-01-15.rollback.sql                      │
│ Scope: Undo ALL rules deployed in one release                       │
│ When: An entire release needs to be backed out                      │
│ How: Execute the release rollback file against BigQuery             │
│                                                                     │
│ This file is a concatenation of individual ROLLBACK files           │
│ for every rule in the release. Built by scripts/build_release.py.   │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ LEVEL 3: Full Table Reset (emergency only)                          │
│ ──────────────────────────────────────────                          │
│                                                                     │
│ File: procedure/Data_Model.sql                                      │
│ Scope: Wipes and recreates all config tables                        │
│ When: Catastrophic corruption or fresh environment setup            │
│ How: Run TRUNCATE on all config tables, then redeploy               │
│      all CONFIG files from the repo in dependency order             │
│                                                                     │
│ This is the nuclear option. The repo IS the source of truth,        │
│ so you can always rebuild from scratch.                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### ROLLBACK Files Do NOT Include Expression Rollbacks

Expressions are shared across products. A product's ROLLBACK file should never delete an expression that other products might use. Expression rollbacks live separately in `shared/expressions.rollback.sql` with conditional DELETEs (only delete if no rule references them).

### Why Expression Rollbacks are Strictly Separated

Separating the expression rollback from the release/config rollback is a **highly recommended, best-practice strategy**. 
Because expressions are shared natively, multiple rules across different releases might rely on the same expression. 
* If Release A deploys `EXP_1`, and Release B deploys a new rule that *uses* `EXP_1`.
* If you were to roll back Release A and blindly delete `EXP_1`, Release B would immediately break!
By separating them into `shared/expressions.rollback.sql` and using safe `NOT EXISTS` condition checks, you guarantee that an expression is only deleted if absolutely zero active rules are currently using it.

---

## 16. The Release Management Process

### When You Cut a Release

A release groups multiple product configs that should deploy together. To create a release:

```
Step 1: Decide which products are in this release
  e.g., KK base load + SP daily merge + new shared expression

Step 2: Run the build script:
  python scripts/build_release.py \
    --name "RELEASE-2024-01-15" \
    --configs \
      products/kk/CONFIG-kk-base-load.sql \
      products/sp/CONFIG-sp-daily-merge.sql \
    --shared shared/expressions.sql

Step 3: Script produces:
  releases/RELEASE-2024-01-15.sql           ← Final Idempotent Deployment File
  releases/RELEASE-2024-01-15.rollback.sql  ← Final Rollback File

### How `build_release.py` Bakes Idempotency into the Release File
The pipeline and `build_release.py` script guarantee idempotency structurally through two approaches:

1. **For Config Updates (The Teardown-then-Rebuild Pattern):** `build_release.py` does not simply concatenate `INSERT` statements. For every `CONFIG` passed to it, the script automatically pulls the corresponding `DELETE` statements from its paired `ROLLBACK` file and places them at the **very top** of the `RELEASE-****.sql` file. 
   *Result:* When BigQuery executes the release file, it first wipes the target `rule_ids` from all tables, and then executes the `INSERT` statements below it. This guarantees zero duplicates and a perfectly clean state.
   
2. **For Shared Expressions (The MERGE Pattern):** Since `shared/expressions.sql` grows continuously, blindly appending it to the release file would create massive duplication. When `build_release.py` reads `shared/expressions.sql`, it programmatically parses and converts every standard `INSERT` statement into a `MERGE ... WHEN NOT MATCHED THEN INSERT` block. These MERGE blocks are injected into the release file. This natively evaluates what is currently active in BigQuery and dynamically skips inserts for expressions that already exist.

Step 4: Commit and push the release files.
```

### Release File Structure

The release file combines individual configs in dependency order. The release rollback combines individual rollbacks in reverse order.

---

## 17. Branch Strategy and PR Rules

### Branch Naming & Environments

**Permanent Environment Branches:**
- `Develop` (Development & Integration)
- `sit` (System Integration Testing)
- `uat` (User Acceptance Testing)
- `preprod` (Pre-production testing)
- `main` (Production)

**Feature & Fix Branches:**
Mandatory Jira ticket prefix for all temporary branches.
- Format: `feature/PBMIS-###-short-description`
- Format: `fix/PBMIS-###-short-description`
- Example: `feature/MIS-452-npk-monthly-load`

### The Promotion Path (Branch-to-Branch)
1. `feature/PBMIS-###` → PR into `Develop` (Review by peer)
2. `Develop` → PR into `sit` (Review by peer)
3. `sit` → PR into `uat` (Review by peer)
4. `uat` → PR into `preprod` (Review by release team)
5. `preprod` → PR into `main` (Production deploy)

### PR Rules

```text
Rule 1: Every change goes through a PR. No direct pushes to any environment branch.
Rule 2: At least 2 approvals required before merge.
Rule 3: Validation pipeline must pass before any feature branch is merged into Develop.
Rule 4: PR title must include Jira ticket: "[PBMIS-###] type(product): description"
```

---

## 18. Environment Strategy

```text
┌─────────────┬──────────────────────┬──────────────────────────────┐
│ Environment │ Target Branch        │ BigQuery Project             │
├─────────────┼──────────────────────┼──────────────────────────────┤
│ DEV         │ Develop              │ {your-project}-dev           │
│ SIT         │ sit                  │ {your-project}-sit           │
│ UAT         │ uat                  │ {your-project}-uat           │
│ PREPROD     │ preprod              │ {your-project}-preprod       │
│ PROD        │ main                 │ {your-project}-prod          │
└─────────────┴──────────────────────┴──────────────────────────────┘
```

The CI/CD pipeline uses **GitHub Environments**. The single `deploy-env.yml` file uses the current branch name to dynamically load the correct environment context.

```yaml
# Conceptual logic in deploy-env.yml:
environment: ${{ github.ref_name }} # e.g. 'dev', 'sit'
env:
  PROJECT_ID: ${{ vars.BQ_PROJECT_ID }}
```

You must configure 5 Environments in GitHub (Settings > Environments), and set the corresponding variables and secrets for each:
- `BQ_PROJECT_ID` (Variable — e.g. 'my-project-sit')
- `GCP_SA_CREDENTIALS` (Secret — specific to that environment)


---

## 19. Day-by-Day Walkthrough: New Rule Lifecycle

```
DAY 1: New Requirement
─────────────────────────────────────────────────────────
Developer gets a request: "We need NPK Monthly (Ticket: MIS-452)
loaded daily as a MERGE into the risk warehouse."

Developer creates a branch:
  git checkout -b feature/MIS-452-npk-monthly

Developer checks: does products/npk/ exist?
  No → he creates it and adds a row to PRODUCT_REGISTRY.md.

Business Analyst writes the user story (with rule_id from business):
  products/npk/US-npk-monthly.md


DAY 1: Generate Files
─────────────────────────────────────────────────────────
APPROACH A (Developer prefers VS Code):
  Opens Copilot Chat → generates LLD → saves file
  Continues chat → generates CONFIG + ROLLBACK → saves both files

OR APPROACH B (if Workspace is available):
  Opens GitHub Issue → describes task in Workspace
  Copilot proposes files → Developer reviews → creates PR

Result: 4 files in products/npk/


DAY 1: Push and PR
─────────────────────────────────────────────────────────
Developer pushes and opens PR:
  "[MIS-452] feat(npk): add NPK classification daily merge config"

Validation pipeline runs automatically:
  ✅ File naming correct
  ✅ SQL syntax valid
  ✅ CONFIG and ROLLBACK match
  ✅ Rule ID NPK_DAILY_CLASS is unique
  ✅ Product folder matches file prefix


DAY 1-2: Review
─────────────────────────────────────────────────────────
You open the PR and review:
  1. US file — requirements clear? rule_id correct?
  2. LLD file — join logic correct? column mapping accurate?
  3. CONFIG file — table names right? constraints respected?
  4. ROLLBACK file — covers all CONFIG tables?

Optional: tag @github-copilot in a PR comment for AI review:
  "@github-copilot review this config against the constraints
   in docs/CONFIG_FILLING_GUIDE.md"

You approve the PR.


DAY 2: Merge and Promote
─────────────────────────────────────────────────────────
Developer merges the PR into the `Develop` branch.

Deployment pipeline runs:
  1. Resolves environment to DEV based on branch name.
  2. Deploys CONFIG to BigQuery DEV project.
  3. Smoke test: CALL ds_cil_bq_code_base.SP_DYNAMIC_ROUTINE_GENERATOR(...)
  4. ✅ Success in DEV.

When testing in DEV is complete, Developer promotes to SIT:
  1. Opens PR comparing `Develop` into `sit`.
  2. PR gets approved by QA.
  3. PR is merged.
  4. Pipeline runs automatically for `sit` branch, deploying to SIT project.

Developer repeats this branch-to-branch promotion process for `uat` and `preprod` until finally opening a PR from `preprod` into `main`. Once merged into `main`, the exact same pipeline runs using PROD environment secrets!

Developer's Airflow DAG in PROD can now call:
  CALL ds_cil_bq_code_base.SP_DYNAMIC_ROUTINE_GENERATOR(
    'NPK_DAILY_CLASS', CURRENT_DATE(), 'airflow_npk_001'
  );


DAY 5: Problem Discovered
─────────────────────────────────────────────────────────
Risk team reports: "Duplicates in the output."

OPTION A — Quick rollback:
  Run products/npk/ROLLBACK-npk-classification.sql against PROD.
  Rule is removed. DAG will fail with "Config not found."
  Developer investigates and fixes.

OPTION B — Fix forward:
  Developer creates branch: fix/npk-classification-join-type
  Updates the CONFIG file (change LEFT to INNER JOIN).
  Opens new PR → review → merge → deploy.

Either way, the fix is tracked in Git with full audit trail.
```

---

## 20. Implementation Phases

### Phase 1: Repository Setup (Week 1-2)

**What you build:**
- Create the repository with the full folder structure.
- Move all existing files (procedure, docs, examples) into the repo.
- Create PRODUCT_REGISTRY.md with your current products.
- Create .github/copilot-instructions.md.
- Write prompt templates (us-to-lld.md, lld-to-config.md).
- Write one complete example manually (US → LLD → CONFIG → ROLLBACK) for one existing product as a reference.
- Write README.md with getting started instructions.

**How you work after Phase 1:**
- You use Copilot Chat to generate LLD + CONFIG + ROLLBACK.
- You save files in the correct product folder.
- You create PRs manually.
- You deploy to BigQuery manually (using the existing process).

**Value gained:** Everything is in Git. Every change is versioned. You have a reference example to follow.

### Phase 2: Validation Pipeline (Week 3-4)

**What you build:**
- validate_config.py script (all 6 checks from Section 13).
- validate-pr.yml GitHub Action (runs on every PR).
- Branch protection rules (require PR, require validation pass, require approval).

**How you work after Phase 2:**
- Same as Phase 1, but now every PR is automatically validated.
- You cannot merge a PR with naming violations, missing files, or invalid SQL.
- You still deploy manually.

**Value gained:** Automated quality gate. Bad configs never reach BigQuery.

### Phase 3: Deployment Pipeline (Week 5-7)

**What you build:**
- deploy.py script (connects to BigQuery, executes SQL).
- deploy-config.yml GitHub Action (runs on merge to main).
- Service account setup for BigQuery DEV and PROD.
- Manual approval gate for PROD deployment.
- Notification setup (Slack/Teams/email on success/failure).

**How you work after Phase 3:**
- You use Copilot Chat (or Workspace) to generate files.
- You push and open PR.
- Pipeline validates automatically.
- You review and merge.
- Pipeline deploys to DEV automatically.
- You approve PROD deployment.
- If it fails, pipeline runs ROLLBACK automatically.

**Value gained:** No more manual SQL execution. Full automation from merge to production.

### Phase 4: Release Management (Week 8-9)

**What you build:**
- build_release.py script.
- Release file generation process.
- Git tagging strategy for releases.
- Manual rollback GitHub Action (trigger from UI to rollback a release).

**Value gained:** Enterprise-grade release management. One-click rollback for entire releases.

---

## 21. Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│ QUICK REFERENCE: How to Add a New ETL Rule                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ 1. git checkout -b feature/{product}-{process}                 │
│                                                                 │
│ 2. Create products/{product}/ folder (if new product)          │
│    Update docs/PRODUCT_REGISTRY.md                             │
│                                                                 │
│ 3. Write: products/{product}/US-{product}-{process}.md         │
│    (include metadata header with rule_id)                      │
│                                                                 │
│ 4. Generate via Copilot Chat:                                  │
│    LLD-{product}-{process}.md                                  │
│    CONFIG-{product}-{process}.sql                              │
│    ROLLBACK-{product}-{process}.sql                            │
│                                                                 │
│ 5. git add → git commit → git push                             │
│    Open Pull Request targeting the `dev` branch                │
│                                                                 │
│ 6. Wait for validation ✅                                       │
│    Get PR approved                                             │
│    Merge into `dev` (auto-deploys to DEV environment)          │
│                                                                 │
│ 7. Promote through environments by opening PRs                 │
│    (`dev` → `sit` → `uat` → `preprod` → `main`)                │
│                                                                 │
│ DONE. Your rule is live.                                       │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│ ROLLBACK: Execute the ROLLBACK file against BigQuery           │
│ RELEASE ROLLBACK: Execute releases/RELEASE-*.rollback.sql     │
├─────────────────────────────────────────────────────────────────┤
│ NAMING: {TYPE}-{product}-{process}.{ext}                       │
│ TYPES: US (.md) | LLD (.md) | CONFIG (.sql) | ROLLBACK (.sql) │
│ FOLDERS: lowercase-with-hyphens (e.g., kk, npk, cust-360)    │
│ RULE IDs: UPPER_SNAKE_CASE from US metadata header            │
└─────────────────────────────────────────────────────────────────┘
```

---

*This document represents the agreed-upon design after alignment discussions. Implementation begins with Phase 1.*
