# /sqe-kit:gen-manual-tests — Generate Manual Test Cases from User Story

**Manual test case generation only.** Accepts a user story, runs the full analysis → generation
→ evaluation → gap-filling pipeline, and produces formatted test case documents (BDD `.feature`
files and/or non-BDD TC markdown files). Does NOT generate automation artifacts.

To also generate automation artifacts in the same run, use `/sqe-kit:e2e` instead.

## Arguments

| Argument | Description |
|----------|-------------|
| `--story <path\|inline>` | **Required.** Path to a user story file, or inline user story text |
| `--project <name>` | Override project name (default: `sqe-project`) |
| `--style bdd\|nonbdd\|both` | TC output format (default: `both`) |
| `--types <list>` | Comma-separated test types: `positive,negative,edge,security,api,ui,integration,performance` (default: auto) |
| `--mode 1\|2\|3\|4` | Orchestrator mode (default: 1 = full workflow with evaluation + gap-filling) |
| `--dry-run` | Preview planned files without writing any |

### Mode Reference

| Mode | Phases | Use when |
|------|--------|---------|
| 1 (default) | Analysis + Generation + Evaluation + Gap-Filling + Formatting | First run — recommended |
| 2 | Analysis + Generation + Formatting | Quick run; skip quality evaluation |
| 3 | Evaluation + Gap-Filling + Formatting | Scenarios already exist in `01-scenarios/`; re-evaluate |
| 4 | Single phase (specify which) | Targeted re-run |

### Examples

```bash
# Full workflow — generates BDD + non-BDD (default)
/sqe-kit:gen-manual-tests --story user-stories/checkout.md

# Non-BDD only — traditional TC format
/sqe-kit:gen-manual-tests --story user-stories/login.md --style nonbdd

# Both formats — BDD .feature + non-BDD .md
/sqe-kit:gen-manual-tests --story user-stories/payment.md --style both

# Quick generation (skip evaluation) — only positive and security tests
/sqe-kit:gen-manual-tests --story user-stories/auth.md --mode 2 --types positive,negative,security

# Preview what would be generated
/sqe-kit:gen-manual-tests --story user-stories/checkout.md --dry-run

# Re-evaluate existing scenarios (mode 3)
/sqe-kit:gen-manual-tests --project my-project --mode 3

# Custom project name
/sqe-kit:gen-manual-tests --story user-stories/checkout.md --project ecommerce-app
```

---

## Execution Pipeline

### Step 0: Setup

- `projectName` = `--project` argument OR `"sqe-project"` (default)
- `outputDir`   = `./outputs` (fixed)
- `testStyle`   = `--style` argument OR `"both"` (default)
- `outputBase`  = `<outputDir>/<projectName>/manual-tests/`
- If `--dry-run`: announce dry-run mode

### Step 1: Parse User Story

Resolve `--story` input using these rules:

| Input | How handled |
|-------|-------------|
| Inline text (contains "As a", "I want", "Feature:", or "Acceptance Criteria") | Used as-is |
| `.md` / `.story.md` / `.txt` | Read file as plain text |
| `.pdf` | Extract text content page-by-page; strip headers/footers |
| `.yml` / `.yaml` | Parse as YAML; look for keys `story`, `userStory`, `feature`, `acceptanceCriteria`; serialize to text block before parsing |
| File path that does not exist | Error: `"User story file not found at [path]"` |
| Unrecognized extension | Attempt to read as plain text; warn if content looks binary |

Apply `skills/user-story-parser.md` (resolve via PLUGIN_DIR):
- Read resolved input text
- Extract: `featureName`, `featureSlug`, `acceptanceCriteria`, `technicalContext`
- Report: "User story: [featureName] | [N] acceptance criteria"

If `--mode 3` (evaluation only): skip Step 1 — scenarios already exist.

### Step 2: Run Master Orchestrator

Read and execute `agents/capability-2/orchestration/quality-master-orchestrator.agent.md` (resolve via PLUGIN_DIR).

Pass:
```
userStoryText       : full text from --story
projectName         : resolved project name
outputBase          : <outputDir>/<projectName>/manual-tests/
testStyle           : resolved testStyle
testTypes           : --types (or "auto")
mode                : --mode (1|2|3|4)
handoffToAutomation : false
dryRun              : --dry-run
```

Phases run by master orchestrator based on `--mode`:

| Mode | Phase 0 | Phase 1 | Phase 2 | Phase 3 |
|------|---------|---------|---------|---------|
| 1 | ✓ Analysis | ✓ Generation | ✓ Evaluation | ✓ Gap-filling |
| 2 | ✓ Analysis | ✓ Generation | — | — |
| 3 | — | — | ✓ Evaluation | ✓ Gap-filling |
| 4 | User specifies | | | |

### Step 3: Format Manual Test Cases

Apply TC formatter skills based on `testStyle`:

**`testStyle` includes `"bdd"`** → apply `skills/tc-formatter-bdd.md` (resolve via PLUGIN_DIR):
- Merge Phase 1 + Phase 3 scenarios
- Apply kit tagging rules (from `rules/bdd-gherkin.md`)
- Assign TC-NNN IDs
- Save: `manual-tcs/bdd/<featureSlug>.feature` + `TC-INDEX.md`

**`testStyle` includes `"nonbdd"`** → apply `skills/tc-formatter-nonbdd.md` (resolve via PLUGIN_DIR):
- Convert Gherkin → TC-ID / Steps / Expected format
- Save: `manual-tcs/nonbdd/<featureSlug>.md` + `TC-INDEX.md`

### Step 4: Quality Lint (always)

Apply rules from `rules/manual-tc-quality.md` (resolve via PLUGIN_DIR):
- Check all TCs have TC IDs
- Check all BDD scenarios have required tags
- Check expected results are specific (not vague)
- Report any warnings inline

### Step 5: Summary

```
✅ Manual Test Case Generation Complete

Feature: [featureName]
Project: [projectName]
Output: outputs/[projectName]/manual-tests/

📋 Generated Files:
   Phase 0 — Analysis:     outputs/[projectName]/manual-tests/00-analysis/
   Phase 1 — Scenarios:    outputs/[projectName]/manual-tests/01-scenarios/  ([N] scenarios)
   Phase 2 — Evaluation:   outputs/[projectName]/manual-tests/02-evaluation/ (score: [N]/100)
   Phase 3 — Gap-filled:   outputs/[projectName]/manual-tests/03-gap-filled/ ([N] added)
   Final TCs (BDD):        outputs/[projectName]/manual-tests/manual-tcs/bdd/
   Final TCs (Non-BDD):    outputs/[projectName]/manual-tests/manual-tcs/nonbdd/

📊 Coverage:
   Test Cases: [N] total ([N] positive, [N] negative, [N] edge, [N] security, ...)
   Quality Score: [N]/100
   Acceptance Criteria Covered: [N]/[total] ([N]%)

💡 Next Steps:
   - Review TCs in manual-tcs/ and share with stakeholders
   - Address [N] context-required gaps (see 02-evaluation/completeness-analysis.json)
   - Generate automation: /sqe-kit:e2e --story [path] --url [url]
     OR: /sqe-kit:generate-automation-tests --story [path] --url [url]
```

---

## Important Rules

- **This command never touches automation pipeline** — no intake-agent, no locator-agent, no POM generation
- Always run `user-story-parser` skill before dispatching to master orchestrator
- Pass complete user story text to all agents — never a summary
- `--dry-run` prints planned file list and exits — no files written
- If `--mode 3` and `01-scenarios/` does not exist: error "No scenarios found. Run mode 1 or 2 first."
