# SQE Kit — User Guide

**Version**: 1.0.0
**Plugin path**: `C:\Users\som20\.claude\plugins\sqe-kit`

---

## What Is SQE Kit?

SQE Kit is a Claude Code plugin that acts as a **Synthetic Quality Engineer** — it takes your
requirements, user stories, or test cases and generates complete, production-ready testing
artifacts through a 3-capability pipeline:

```
Capability 1 ─ Requirement Doc  →  User Stories
Capability 2 ─ User Story       →  Manual Test Cases (BDD + Non-BDD)
Capability 3 ─ Manual TCs       →  Playwright TypeScript Automation
```

---

## Installation

### On This Machine

Already installed at `C:\Users\som20\.claude\plugins\sqe-kit`.

### On Any Other Machine

**Step 1 — Copy the plugin folder**

You have two options:

**Option A — Git repository (recommended)**
```bash
# 1. Create a git repo from the plugin folder
cd C:\Users\som20\.claude\plugins\sqe-kit
git init
git add .
git commit -m "Initial SQE Kit plugin"

# Push to GitHub / GitLab / Bitbucket / private git server
git remote add origin https://github.com/your-org/sqe-kit.git
git push -u origin main
```

On the target machine:
```bash
# Clone into the plugins directory
git clone https://github.com/your-org/sqe-kit.git "%USERPROFILE%\.claude\plugins\sqe-kit"
# Mac/Linux:
# git clone https://github.com/your-org/sqe-kit.git ~/.claude/plugins/sqe-kit
```

**Option B — Manual copy (zip/USB/network share)**
```
Copy the entire sqe-kit folder to:
  Windows : C:\Users\<username>\.claude\plugins\sqe-kit\
  Mac/Linux: ~/.claude/plugins/sqe-kit/
```

**Step 2 — Update PLUGIN_DIR in settings.json**

On the target machine, open `sqe-kit/settings.json` and update every path from
`C:\\Users\\som20\\.claude\\plugins\\sqe-kit` to the actual install path on that machine.

**Windows example**:
```json
"sqePluginDir": "C:\\Users\\jane\\.claude\\plugins\\sqe-kit"
```
Also update the `defaultSystemPrompt` path references in the same file.

> **Tip**: If you maintain the plugin in a git repo, create a simple
> `setup.ps1` / `setup.sh` script that replaces the username automatically.

### Do You Need the Marketplace?

**No — for personal or team use, git distribution is sufficient and preferred.**

For broader public distribution, Anthropic provides a plugin submission process at
`claude.ai/settings/plugins`. For team-internal use, a private git repository is the
standard approach — no marketplace listing required.

---

## Starting a Session

```bash
# From any project directory
claude --plugin-dir C:\Users\som20\.claude\plugins\sqe-kit
```

**What Claude says on startup:**

- *If no project fingerprint exists* (new project / greenfield):
  `"No project fingerprint found. Run /sqe-kit:scan-project to scan this project..."`

- *If `.sqe/project-fingerprint.json` exists* (existing Playwright project / brownfield):
  `"Active project: my-app | nonbdd | POMs: src/pages/ | Tests: tests/nonbdd/"`

---

## The 5 Commands

### 1. `/sqe-kit:scan-project` — Enable Brownfield Mode

Scans an existing Playwright TypeScript project and writes `.sqe/project-fingerprint.json`.
Run this **once** per existing project before any generation commands.

```bash
/sqe-kit:scan-project
/sqe-kit:scan-project --project /path/to/project   # scan a specific directory
/sqe-kit:scan-project --dry-run                    # preview without writing
```

**What it detects**: POM directories, spec directories, feature files, import aliases,
constructor patterns, locator styles, test style (bdd/nonbdd/both), all existing files.

**Output**: `.sqe/project-fingerprint.json` in the project root.

---

### 2. `/sqe-kit:gen-user-stories` — Requirement Doc → User Stories

Takes any requirement document (PDF, Word, Markdown, image, etc.) and generates structured
user story files ready for manual TC generation.

```bash
# Single requirement document
/sqe-kit:gen-user-stories --req docs/prd.pdf

# With doc type hint
/sqe-kit:gen-user-stories --req docs/feature-spec.md --type feature

# Multiple files combined
/sqe-kit:gen-user-stories --req "docs/brd.pdf,docs/api-contract.yaml"

# Entire directory
/sqe-kit:gen-user-stories --req docs/requirements/

# Generate stories + manual TCs in one shot
/sqe-kit:gen-user-stories --req docs/prd.pdf --then-manual-tests --style both

# Preview without writing
/sqe-kit:gen-user-stories --req docs/prd.pdf --dry-run
```

**Supported formats**: `.pdf`, `.docx`, `.md`, `.txt`, `.yaml`, `.json`, `.xml`,
`.bpmn`, `.csv`, `.xlsx`, `.eml`, `.png`, `.jpg`

**Output** → `outputs/<project>/user-stories/`
```
US-001-login.md
US-002-checkout.md
US-INDEX.md
00-analysis/req-analysis.json
01-features/feature-map.json
```

| Argument | Description |
|---|---|
| `--req <path>` | Required. File, comma-separated list, or directory |
| `--project <name>` | Override project name (default: `sqe-project`) |
| `--type <doctype>` | Hint: `brd`, `prd`, `feature`, `epic`, `workflow`, `api`, `ux`, `meeting`, `email`, `ticket`, `other` |
| `--limit <n>` | Max stories to generate (default: 20) |
| `--output-mode auto\|single\|separate` | File grouping strategy |
| `--then-manual-tests` | Also run manual TC generation after stories |
| `--style bdd\|nonbdd\|both` | TC format when using `--then-manual-tests` |
| `--dry-run` | Preview without writing |

---

### 3. `/sqe-kit:gen-manual-tests` — User Story → Manual Test Cases

Takes a user story and runs a 5-phase quality pipeline to generate comprehensive, tagged
BDD `.feature` files and/or Non-BDD TC markdown files.

```bash
# Full pipeline — BDD + Non-BDD (default)
/sqe-kit:gen-manual-tests --story inputs/user-stories/login.md

# Non-BDD only
/sqe-kit:gen-manual-tests --story inputs/user-stories/login.md --style nonbdd

# Specific test types only
/sqe-kit:gen-manual-tests --story inputs/user-stories/auth.md --types positive,negative,security

# Quick mode — skip quality evaluation
/sqe-kit:gen-manual-tests --story story.md --mode 2

# Inline user story
/sqe-kit:gen-manual-tests --story "As a shopper I want to add items to cart..."

# Custom project name
/sqe-kit:gen-manual-tests --story story.md --project ecommerce-app
```

**The 5-Phase Pipeline:**

| Phase | What runs | Output |
|---|---|---|
| 0 | user-story-analyzer | `00-analysis/test-type-analysis.json` |
| 1 | 8 specialist agents (positive, negative, edge, integration, security, performance, api, ui) | `01-scenarios/*.feature` |
| 2 | quality-evaluator | `02-evaluation/completeness-analysis.json` + score |
| 3 | gap-filler | `03-gap-filled/` (fills auto-fillable gaps) |
| 4 | tc-formatter | `manual-tcs/bdd/*.feature` + `manual-tcs/nonbdd/*.md` |

**Modes:**

| Mode | Phases | When to use |
|---|---|---|
| 1 (default) | All 5 phases | First run — recommended |
| 2 | Phases 0–1–4 only | Quick run, skip evaluation |
| 3 | Phases 2–3–4 only | Re-evaluate existing scenarios |

**Output** → `outputs/<project>/manual-tests/`

| Argument | Description |
|---|---|
| `--story <path\|inline>` | Required. File path or inline text |
| `--project <name>` | Override project name |
| `--style bdd\|nonbdd\|both` | TC output format (default: `both`) |
| `--types <list>` | `positive,negative,edge,security,api,ui,integration,performance` |
| `--mode 1\|2\|3\|4` | Pipeline mode |
| `--dry-run` | Preview without writing |

---

### 4. `/sqe-kit:generate-automation-tests` — Test Cases → Automation

Accepts test cases (or a user story) and generates locators, Page Objects, and test specs
for Playwright TypeScript.

```bash
# From test cases + DOM snapshot
/sqe-kit:generate-automation-tests --cases "TC-001: Login with valid credentials" --dom inputs/snapshots/

# From test cases + live URL
/sqe-kit:generate-automation-tests --cases inputs/test-cases/login.md --url https://staging.myapp.com

# From user story (generates manual TCs first, then automation)
/sqe-kit:generate-automation-tests --story inputs/user-stories/login.md --url https://app.com

# BDD artifacts
/sqe-kit:generate-automation-tests --cases login.md --dom inputs/snapshots/ --bdd

# Brownfield — extend existing project
/sqe-kit:generate-automation-tests --cases new-tcs.md --url https://app.com --extend
```

**Pipeline steps:**
1. intake-agent — parses test cases, detects brownfield/greenfield
2. locator-agent — extracts selectors (Tier 1: live app / Tier 2: DOM / Tier 3: AI-guided)
3. pom-agent — generates Page Object files
4. testgen-agent / bdd-agent — generates `.spec.ts` or `.feature` + `.steps.ts`
5. reporting-agent — configures Allure + HTML reporters

**Output** (greenfield) → `outputs/<project>/src/pages/` + `outputs/<project>/tests/`

| Argument | Description |
|---|---|
| `--cases <path\|inline>` | Test case descriptions |
| `--story <path\|inline>` | User story (runs TC generation first) |
| `--url <url>` | Live app URL (Tier 1 locators) |
| `--dom <path>` | DOM snapshot dir or `.html` file (Tier 2 locators) |
| `--bdd` | Also generate BDD artifacts |
| `--project <name>` | Override project name |
| `--extend` | Append to existing files (brownfield) |
| `--overwrite` | Overwrite existing files (brownfield) |
| `--dry-run` | Preview without writing |

---

### 5. `/sqe-kit:e2e` — Full Pipeline (Req Doc OR User Story → Everything)

The one-command pipeline. Start from a requirement doc or user story and get manual TCs
+ full automation artifacts in a single run.

```bash
# Full pipeline from requirement doc
/sqe-kit:e2e --req docs/prd.pdf --url https://staging.myapp.com

# Full pipeline from user story
/sqe-kit:e2e --story inputs/user-stories/checkout.md --url https://app.com

# Manual TCs only (no automation)
/sqe-kit:e2e --story inputs/user-stories/login.md --skip-automation

# BDD style, specific test types
/sqe-kit:e2e --story story.md --style bdd --types positive,negative,security --url https://app.com

# Brownfield — extend existing project
/sqe-kit:e2e --story new-feature.md --url https://app.com --extend

# With DOM snapshots instead of live URL
/sqe-kit:e2e --story story.md --dom inputs/snapshots/

# Req doc — manual TCs only, no automation
/sqe-kit:e2e --req docs/spec.md --skip-automation
```

| Argument | Description |
|---|---|
| `--req <path>` | Requirement doc input. Mutually exclusive with `--story`. |
| `--story <path\|inline>` | User story input. Mutually exclusive with `--req`. |
| `--url <url>` | Live app URL for Tier 1 locator extraction |
| `--dom <path>` | DOM snapshot dir/file for Tier 2 locator extraction |
| `--style bdd\|nonbdd\|both` | TC and automation format (default: from kit.config.json) |
| `--types <list>` | Comma-separated test types to generate |
| `--mode 1\|2\|3\|4` | Manual TC orchestrator mode (default: 1) |
| `--skip-automation` | Stop after manual TCs — no automation pipeline |
| `--extend` | Append to existing files (brownfield) |
| `--overwrite` | Overwrite existing files (brownfield) |
| `--project <name>` | Override project name |
| `--type <doctype>` | Doc type hint when using `--req` |
| `--limit <n>` | Max user stories from req doc (default: 20) |
| `--dry-run` | Preview all planned files, write nothing |

---

## Greenfield vs Brownfield Mode

### Greenfield (New Project)

- **When**: no `.sqe/project-fingerprint.json` in the working directory
- **Behavior**: All generated files go into `outputs/<projectName>/`
- **POMs** → `outputs/<project>/src/pages/`
- **Specs** → `outputs/<project>/tests/nonbdd/` or `tests/bdd/`
- **No conflict checking** — writes freely

### Brownfield (Existing Playwright Project)

- **When**: `.sqe/project-fingerprint.json` exists
- **Activate**: Run `/sqe-kit:scan-project` once in the project root
- **Behavior**: Generated files go into the project's existing directories (e.g., `src/pages/`, `tests/`)
- **Conflict handling**:

| Scenario | Default | With `--extend` | With `--overwrite` |
|---|---|---|---|
| POM already exists | Skip | Append new methods | Replace file |
| Spec already exists | Skip | Append new tests | Replace file |
| Feature file exists | Skip | Append scenarios | Replace file |

- **Inherits**: `importAlias`, `constructorPattern`, `locatorStyle`, naming conventions
  from the scanned fingerprint — so generated code matches your existing codebase style.

---

## Locator Strategy Tiers

| Tier | When | Selector confidence |
|---|---|---|
| 1 — Playwright MCP | `--url` provided | Up to 0.99 |
| 2 — DOM Snapshot | `--dom` provided, no `--url` | Up to 0.95 |
| 3 — AI-Guided | Neither `--url` nor `--dom` | Capped at 0.75 |

> If using Tier 3, Claude will warn you that locators are inferred and must be
> verified against the real application.

---

## Integration Plugins

After any generation command completes, Claude checks `configs/integrations.yml`.
Enable integrations by editing that file:

```yaml
# C:\Users\som20\.claude\plugins\sqe-kit\configs\integrations.yml
plugins:
  github:
    enabled: false     # set true → commits artifacts + optional PR creation
  jira:
    enabled: false     # set true → fetch TCs from Jira, push smoke results back
  allureServer:
    enabled: false     # set true → publish results to Allure server after /run-smoke
```

---

## Input File Conventions

Place input files in these directories inside your project (or provide full paths):

```
<your-project>/
├── inputs/
│   ├── req-docs/        ← requirement documents (for --req)
│   ├── user-stories/    ← user story files (for --story)
│   ├── test-cases/      ← manual TC files (for --cases)
│   └── snapshots/       ← DOM HTML snapshots (for --dom)
└── outputs/             ← all generated artifacts land here
```

---

## Output Structure

```
outputs/<project-name>/
├── user-stories/              ← Capability 1 output
│   ├── 00-analysis/
│   ├── 01-features/
│   ├── US-001-<slug>.md
│   └── US-INDEX.md
├── manual-tests/              ← Capability 2 output
│   ├── 00-analysis/
│   ├── 01-scenarios/
│   ├── 02-evaluation/
│   ├── 03-gap-filled/
│   └── manual-tcs/
│       ├── bdd/               ← Tagged .feature files
│       └── nonbdd/            ← TC-ID / Steps / Expected .md files
├── intake.summary.json        ← Automation pipeline contract
├── selectors.json             ← Selector map (preserve between runs)
├── src/pages/                 ← Page Object Model files
├── tests/
│   ├── nonbdd/                ← Playwright .spec.ts files
│   └── bdd/                   ← .feature + .steps.ts files
└── configs/
    ├── playwright.config.ts
    └── .github/workflows/ui.yml
```

---

## Typical Workflows

### Workflow A — Starting from scratch (greenfield)

```bash
# 1. Start Claude with the plugin
claude --plugin-dir C:\Users\som20\.claude\plugins\sqe-kit

# 2. Generate user stories from your PRD
/sqe-kit:gen-user-stories --req inputs/req-docs/prd.pdf

# 3. Full pipeline for a story (TCs + automation)
/sqe-kit:e2e --story outputs/sqe-project/user-stories/US-001-login.md --url https://staging.app.com

# 4. Review manual TCs in outputs/.../manual-tcs/ then run tests
npx playwright test
```

### Workflow B — Adding tests to an existing project (brownfield)

```bash
# 1. Start Claude from inside your existing Playwright project
cd C:\Projects\my-app
claude --plugin-dir C:\Users\som20\.claude\plugins\sqe-kit

# 2. Scan the project once (only needed once)
/sqe-kit:scan-project

# 3. Generate tests for a new feature, extending existing code
/sqe-kit:e2e --story new-feature.md --url https://staging.app.com --extend
```

### Workflow C — Manual TCs only (no automation)

```bash
/sqe-kit:gen-manual-tests --story inputs/user-stories/checkout.md --style both
# Review outputs/.../manual-tcs/ and share with QA team
```

### Workflow D — Automation from existing test cases

```bash
/sqe-kit:generate-automation-tests \
  --cases inputs/test-cases/payment.md \
  --dom inputs/snapshots/ \
  --project payment-module
```

---

## Sharing the Plugin With Your Team

### Option 1 — Git Repository (Recommended)

```bash
# 1. Initialise a repo in the plugin folder
cd C:\Users\som20\.claude\plugins\sqe-kit
git init && git add . && git commit -m "Initial SQE Kit plugin"

# 2. Push to your org's git host (GitHub, GitLab, Azure DevOps, etc.)
git remote add origin https://github.com/your-org/sqe-kit.git
git push -u origin main

# 3. Each team member installs it once
git clone https://github.com/your-org/sqe-kit.git "%USERPROFILE%\.claude\plugins\sqe-kit"
```

Then create a one-liner setup script for your team:

```powershell
# setup.ps1 — run after cloning
$pluginPath = "$env:USERPROFILE\.claude\plugins\sqe-kit"
(Get-Content "$pluginPath\settings.json") `
  -replace 'som20', $env:USERNAME | `
  Set-Content "$pluginPath\settings.json"
Write-Host "SQE Kit ready. Run: claude --plugin-dir $pluginPath"
```

### Option 2 — Claude Plugin Marketplace (Public Distribution)

If you want to list it publicly so anyone can discover it, Anthropic provides a plugin
submission process. Visit `claude.ai/settings/plugins` (requires Anthropic account) to
submit. Once listed, any Claude Code user can install it via:

```bash
/plugin install sqe-kit
```

For team-internal use you do **not** need the marketplace — git distribution is sufficient.

---

## Quick Reference Card

| Goal | Command |
|---|---|
| Enable brownfield mode | `/sqe-kit:scan-project` |
| Req doc → User stories | `/sqe-kit:gen-user-stories --req <path>` |
| User story → Manual TCs | `/sqe-kit:gen-manual-tests --story <path>` |
| Manual TCs → Automation | `/sqe-kit:generate-automation-tests --cases <path> --url <url>` |
| Everything in one run | `/sqe-kit:e2e --story <path> --url <url>` |
| Manual TCs only (no automation) | `/sqe-kit:e2e --story <path> --skip-automation` |
| Preview without writing | Add `--dry-run` to any command |
| Add tests to existing project | Add `--extend` to any generation command |

---

## Troubleshooting

**"No project fingerprint found"**
→ This is informational in a new project (greenfield). Run `/sqe-kit:scan-project` only
if you're working inside an existing Playwright project.

**"Locators are AI-guided (score capped at 0.75)"**
→ Provide `--url` (live app) or `--dom` (HTML snapshots) to get higher-confidence selectors.

**Existing files being skipped in brownfield mode**
→ Use `--extend` to append new tests, or `--overwrite` to replace. Default is always skip.

**TypeScript errors after generation**
→ The pipeline runs `npx tsc --noEmit` automatically. If errors remain, check import paths
in generated specs — they should match your project's `tsconfig.json` `paths` aliases.

**Plugin path not found on a new machine**
→ Check that `settings.json`'s `defaultSystemPrompt` and `sqePluginDir` both reflect the
correct install path for that machine.
