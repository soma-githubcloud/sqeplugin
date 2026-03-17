# /sqe-kit:e2e — Full Pipeline: Req Doc OR User Story → Manual TCs → Automation Artifacts

**End-to-end command.** Accepts either a **requirement document** (`--req`) or a **user story**
(`--story`) as input. Generates manual test cases (BDD or non-BDD), then optionally feeds them
into the automation generation pipeline.

Works in both **greenfield** (new projects) and **brownfield** (existing Playwright TS projects)
mode. Brownfield mode is activated automatically when `.sqe/project-fingerprint.json` exists.
Run `/sqe-kit:scan-project` first to enable brownfield mode.

- `--req`: runs req-to-story pipeline first, then processes each generated user story through
  the full manual TC → automation chain
- `--story`: skips req-to-story; goes directly to manual TC generation

## Arguments

| Argument | Description |
|----------|-------------|
| `--req <path>` | Path to a requirement document (BRD, PRD, Feature Spec, etc.). Mutually exclusive with `--story`. When provided, req-to-story pipeline runs first. |
| `--story <path\|inline>` | Path to a user story file, or inline user story text. Mutually exclusive with `--req`. |
| `--project <name>` | Override project name from `kit/kit.config.json` |
| `--style bdd\|nonbdd\|both` | Manual TC format AND automation test style (default: from `kit/kit.config.json`) |
| `--url <url>` | Live app URL for Tier-1 locator extraction during automation phase |
| `--dom <path>` | DOM snapshot directory/file for Tier-2 locator extraction |
| `--types <list>` | Comma-separated specialist types to run: `positive,negative,edge,security,api,ui,integration,performance` (default: auto from analyzer) |
| `--mode 1\|2\|3\|4` | Orchestrator mode (default: 1 = full 5-phase workflow with evaluation + gap-filling) |
| `--skip-automation` | Stop after manual TC generation — do not run automation pipeline |
| `--dry-run` | Preview planned files without writing any |
| `--type <doctype>` | Hint doc type when using `--req`: `brd`, `prd`, `feature`, `epic`, `workflow`, `data`, `api`, `ux`, `meeting`, `email`, `ticket`, `other` (auto-detected if omitted) |
| `--limit <n>` | Max user stories to extract from requirement doc (default: 20). Only used with `--req`. |
| `--extend` | Append to existing POM/spec files instead of skipping (brownfield) |
| `--overwrite` | Overwrite existing POM/spec files (brownfield) |

**Input validation**: Error if neither `--req` nor `--story` is provided. Error if both are provided simultaneously.

### `--story` Resolution

Supported input types (all resolved before parsing):

| Input | How handled |
|-------|-------------|
| Inline text (contains "As a", "I want", "Feature:", or "Acceptance Criteria") | Used as-is |
| `.md` / `.story.md` / `.txt` | Read file as plain text |
| `.pdf` | Extract text content page-by-page; strip headers/footers |
| `.yml` / `.yaml` | Parse as YAML; look for keys `story`, `userStory`, `feature`, `acceptanceCriteria`; serialize to text block before parsing |
| File path that does not exist | Error: `"User story file not found at [path]"` |
| Unrecognized extension | Attempt to read as plain text; warn if content looks binary |

### Examples

```bash
# ── Req-doc input ──────────────────────────────────────────────────────────
# Full pipeline: req doc → user stories → manual TCs → automation
/sqe-kit:e2e --req inputs/req-docs/prd.pdf --url https://staging.myapp.com

# Req doc → user stories → manual TCs only (no automation)
/sqe-kit:e2e --req inputs/req-docs/prd.pdf --skip-automation

# Req doc with doc-type hint + BDD style
/sqe-kit:e2e --req inputs/req-docs/feature-spec.docx --type feature --style bdd --skip-automation

# ── User story input ────────────────────────────────────────────────────────
# Full pipeline: user story → manual TCs → automation
/sqe-kit:e2e --story user-stories/checkout.md --url https://app.example.com

# User story → manual TCs only
/sqe-kit:e2e --story user-stories/checkout.md --skip-automation

# Specific test types + both TC formats
/sqe-kit:e2e --story user-stories/login.md --style both --types positive,negative,security

# Inline user story
/sqe-kit:e2e --story "As a shopper I want to add items to cart so that I can purchase them"

# Quick mode (no evaluation/gap-filling)
/sqe-kit:e2e --story user-stories/checkout.md --mode 2 --skip-automation

# Brownfield — extend existing project
/sqe-kit:e2e --story user-stories/new-feature.md --url https://app.example.com --extend
```

---

## Execution Pipeline

### Step 0: Load Kit Context

- Read `kit/kit.config.json` (resolve via PLUGIN_DIR) — get `id`, `tech`, `testStyle`, `outputDir`, `projectName`
- Read `kit/KIT.md` (resolve via PLUGIN_DIR) for naming conventions
- Check if `.sqe/project-fingerprint.json` exists in CWD:
  - If yes: **brownfield mode** — note will be shown during intake-agent
  - If no: **greenfield mode**
- Confirm to user: `Active kit: <id> | <tech.testRunner> | <testStyle>`
- Resolve `projectName`: `--project` argument overrides `kit/kit.config.json`
- Resolve `testStyle`: `--style` argument overrides `kit/kit.config.json`
- Determine input mode: `--req` → set `inputMode = "req"`; `--story` → set `inputMode = "story"`

### Step 0.5: Req-to-Story Phase _(only when `--req` is provided)_

Run the full req-to-story pipeline before manual TC generation.

**5a — Ingest**
Read and execute `agents/capability-1/req-ingestor.agent.md` (resolve via PLUGIN_DIR). Pass:
```
reqInput    : value of --req
projectName : resolved project name
outputBase  : <outputDir>/<projectName>/user-stories/
docTypeHint : value of --type (or null)
dryRun      : --dry-run flag
```
Outputs: `<outputBase>/00-analysis/extraction-manifest.json`

**5b — Analyze**
Read and execute `agents/capability-1/req-analyzer.agent.md` (resolve via PLUGIN_DIR). Pass extraction manifest path.
Outputs: `<outputBase>/00-analysis/req-analysis.json`

**5c — Split into Features**
Read and execute `agents/capability-1/feature-splitter.agent.md` (resolve via PLUGIN_DIR). Pass extraction manifest + req analysis.
Outputs: `<outputBase>/01-features/feature-map.json`

If `--dry-run`: print feature list and stop here.

**5d — Write User Story Files**
Read and execute `agents/capability-1/user-story-writer.agent.md` (resolve via PLUGIN_DIR). Pass feature map + req analysis.
Outputs: `US-NNN-<slug>.md` files + `US-INDEX.md` in `<outputBase>/`

**5e — Apply Quality Lint**
Apply `rules/user-story-quality.md` (resolve via PLUGIN_DIR) across all generated stories.
Report warnings inline; do not fail on warnings.

**5f — Set story loop**
```
storyFiles = [ all generated US-NNN-*.md paths, sorted ]
```

Report:
```
✅ User Stories Generated: [N] stories from [source]
   Location: outputs/[projectName]/user-stories/
   Proceeding to generate manual test cases for each story...
```

Then execute Steps 1–6 for each story file in `storyFiles` (sequentially).
Set `outputBase` = `<outputDir>/<projectName>/manual-tests/<featureSlug>/` per story to avoid collisions.

---

### Step 1: Parse User Story

_(When `--story` provided: `outputBase` = `<outputDir>/<projectName>/manual-tests/`)_

Apply `skills/user-story-parser.md` (resolve via PLUGIN_DIR):
- Read `--story` input (file or inline) — or current story file from the loop in Step 0.5
- Extract: `featureName`, `featureSlug`, `acceptanceCriteria`, `technicalContext`
- Report: "User story loaded: [featureName] | [AC count] acceptance criteria found"
- If `--dry-run`: show parsed fields and planned files, then stop

### Step 2: Manual Test Case Generation (Phase 1–3)

Read and execute `agents/capability-2/orchestration/quality-master-orchestrator.agent.md` (resolve via PLUGIN_DIR).

Pass:
```
userStoryText   : full text from current story
projectName     : resolved project name
outputBase      : <outputDir>/<projectName>/manual-tests/[<featureSlug>/] (per-story subdir when --req)
testStyle       : resolved testStyle
testTypes       : --types value (or "auto")
mode            : --mode value (default: 1)
handoffToAutomation : false
dryRun          : --dry-run value
```

The master orchestrator runs:
- Phase 0: `agents/capability-2/analysis/user-story-analyzer.agent.md` → `00-analysis/`
- Phase 1: `agents/capability-2/orchestration/quality-orchestrator.agent.md` → `01-scenarios/`
- Phase 2 (mode 1 only): `agents/capability-2/orchestration/quality-evaluator.agent.md` → `02-evaluation/`
- Phase 3 (mode 1, if gaps exist): `agents/capability-2/analysis/gap-filler.agent.md` → `03-gap-filled/`

(All agent paths resolve via PLUGIN_DIR)

### Step 3: Format Manual Test Cases (Phase 4)

After master orchestrator completes, apply TC formatter skills:

**If `testStyle` includes `"bdd"`**:
- Apply `skills/tc-formatter-bdd.md` (resolve via PLUGIN_DIR)
- Input: `01-scenarios/` + `03-gap-filled/` (if exists)
- Output: `manual-tcs/bdd/<featureSlug>.feature` + `manual-tcs/bdd/TC-INDEX.md`

**If `testStyle` includes `"nonbdd"`**:
- Apply `skills/tc-formatter-nonbdd.md` (resolve via PLUGIN_DIR)
- Input: same
- Output: `manual-tcs/nonbdd/<featureSlug>.md` + `manual-tcs/nonbdd/TC-INDEX.md`

Report:
```
✅ Manual Test Cases Generated

Project: [projectName]
Feature: [featureName]
Location: outputs/[projectName]/manual-tests/manual-tcs/

BDD (.feature): [N] scenarios in [N] files
Non-BDD (.md): [N] test cases in [N] files
```

### Step 4: Automation Pipeline (unless `--skip-automation`)

Feed formatted manual TCs as input to the automation pipeline.

**Confirm before proceeding** (unless `--dry-run`):
```
Manual TCs are ready. Proceeding to generate automation artifacts.
Locator strategy: [playwright-mcp | dom-based | ai-guided]
```

Run the following agents in order:

**Step 4a: intake-agent**
Read and execute `agents/capability-3/intake-agent.md` (resolve via PLUGIN_DIR). Pass:
- `appUrl`: `--url` value (or null)
- `domFiles`: resolved from `--dom` (or empty)
- `testCasesRaw`: path to formatted manual TCs
  - BDD testStyle → path to `manual-tcs/bdd/<featureSlug>.feature`
  - nonbdd testStyle → path to `manual-tcs/nonbdd/<featureSlug>.md`
  - both → both paths
- `kitConfig`: full `kit/kit.config.json` contents

intake-agent auto-detects brownfield mode via `.sqe/project-fingerprint.json`.

Output: `intake.summary.json` (greenfield: `<outputDir>/<projectName>/`; brownfield: `.sqe/`)

**Step 4b: locator-agent** (if `selectors.json` not already present or user chooses to regenerate)
Read and execute `agents/capability-3/locator-agent.md` (resolve via PLUGIN_DIR) with strategy from `intake.summary.json`.

**Step 4c: pom-agent**
Read and execute `agents/capability-3/pom-agent.md` (resolve via PLUGIN_DIR) → generates POM files.
Pass `intakeSummary` including brownfield context for conflict checks.

**Step 4d: testgen-agent and/or bdd-agent**
Based on `testStyle` from `intake.summary.json`:
- `"nonbdd"` → run `agents/capability-3/testgen-agent.md` (resolve via PLUGIN_DIR) only
- `"bdd"` → run `agents/capability-3/bdd-agent.md` (resolve via PLUGIN_DIR) only
- `"both"` → run both agents

Both agents: pass `intakeSummary` including brownfield context for conflict checks.

**Step 4e: reporting-agent**
Read and execute `agents/capability-3/reporting-agent.md` (resolve via PLUGIN_DIR).
Configure Allure + HTML reporters (skip if already configured).

### Step 5: Validate (if automation ran)

- Run `npx tsc --noEmit` inside project directory
- Run `npx playwright test --list` to confirm test discovery
- Fix unambiguous import path issues automatically

### Step 6: Summary (per story, and consolidated at end)

Print a consolidated summary table:

```
📋 Manual Test Cases
   Location: outputs/[projectName]/manual-tests/manual-tcs/
   BDD: [N] scenarios in [N] .feature files
   Non-BDD: [N] TCs in [N] .md files
   Quality Score: [N]/100 (or "N/A — evaluation skipped")

🤖 Automation Artifacts (if applicable)
   POMs: [N] files in <pomDir>
   Specs: [N] .spec.ts files in <nonbddDir>
   Features: [N] .feature files in <bddDir>
   Steps: [N] .steps.ts files in <stepsDir>

📁 All outputs: <outputDir>/<projectName>/
```

When `--req` was used, print a final cross-story summary after all stories are processed:
```
📦 Full Pipeline Complete
   Source: [req-doc filename]
   Stories processed: [N]
   Total manual TCs: [N]
   Total automation specs: [N]
   Outputs: <outputDir>/<projectName>/
```

---

## Important Rules

- `--req` and `--story` are mutually exclusive — error if both are provided
- When using `--req`, Step 0.5 always runs before any story processing
- Always run `user-story-parser` first — it normalizes the story before any agent sees it
- Always pass the **complete** user story text to specialist agents — never a summary
- Manual TCs are generated BEFORE automation artifacts — they are the input contract
- `--skip-automation` is respected strictly — do not run any automation agents
- If `--url` and `--dom` are both absent and `--skip-automation` is NOT set, warn the user:
  `"No URL or DOM provided. Automation locators will use AI-guided strategy (Tier 3 — scores capped at 0.75). Add --url or --dom for higher-confidence selectors."`
- The canonical selector file is `selectors.json` — never `locators.json`
- If `--dry-run`: print planned file list for ALL steps (req-to-story + manual + automation), write nothing
- When `--req` produces multiple user stories, process them **sequentially** — not in parallel
- Brownfield: default behavior is to skip existing files; use `--extend` or `--overwrite` to change
