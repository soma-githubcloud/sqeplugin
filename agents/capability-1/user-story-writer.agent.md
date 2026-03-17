# user-story-writer — User Story Generation Agent

**Role**: Step 4 of the `/sqe-kit:gen-user-stories` pipeline. Reads `feature-map.json` and generates
one properly formatted user story file per feature, applying quality rules and preserving
traceability back to the source requirement document.

---

## Input Parameters

```
featureMapPath         : string  — path to feature-map.json from feature-splitter
reqAnalysisPath        : string  — path to req-analysis.json (for NFRs + domain)
projectName            : string
outputBase             : string  — outputs/<projectName>/user-stories/ (greenfield) or .sqe/user-stories/ (brownfield)
outputMode             : string  — "auto" | "single" | "separate" (default: "auto")
dryRun                 : boolean
```

---

## User Story File Format

Each generated file follows this exact structure:

```markdown
# US-<NNN>: <Feature Title>

**Source**: `<source filename>` | **Feature ID**: F-<NNN> | **Section**: <sourceSection>
**Generated**: <ISO date> | **Status**: Draft

---

## User Story

As a **<actor/role>**
I want to **<goal — what the user wants to accomplish>**
So that **<benefit — the value or outcome they receive>**

---

## Acceptance Criteria

- **AC-01**: <Specific, testable condition — present tense, active voice>
- **AC-02**: <Another condition>
- **AC-03**: <Another condition>
<!-- minimum 3 ACs; maximum 10 per story -->

---

## Technical Context

| Aspect | Details |
|--------|---------|
| Has UI | Yes / No |
| UI Components | <list if applicable> |
| Has API | Yes / No |
| API Endpoints | <list if applicable> |
| External Systems | <list if applicable> |
| Database | <entities affected, if known> |
| Performance SLAs | <from NFRs, if applicable> |
| Security / Compliance | <from NFRs, if applicable> |
| Accessibility | <from NFRs, if applicable> |

---

## Notes

> **Open Questions** (resolve before development):
> - <question 1 from feature-map openQuestions>
> - <question 2>

> **Constraints**:
> - <constraints from source document>

> **Out of Scope** (for this story):
> - <items explicitly excluded>
```

---

## Execution Steps

### Step 1: Read Inputs

Read `feature-map.json` (features array) and `req-analysis.json` (NFRs, domain, actors).

### Step 2: Process Each Feature

For each feature in `features[]`, generate one user story:

#### 2a. Assign US ID
Sequential: `US-001`, `US-002`, ... (padded to 3 digits minimum)

#### 2b. Write "As a / I want / So that" Statement

Use `feature.actor`, `feature.goal`, `feature.benefit` from feature-map.

If `goal` or `benefit` is missing from feature-map, infer from `feature.description`:
- `goal` = what the actor does with this feature (verb + object)
- `benefit` = why it matters to them (outcome, not implementation)

Rules:
- `As a` → use the specific role, never "user" if a more specific role is available
- `I want to` → action verb + object, user perspective (not system perspective)
- `So that` → business/personal value, not technical outcome

**Anti-patterns to avoid:**
| Bad | Good |
|-----|------|
| "As a user" when role is known | "As a **Patient**" |
| "I want the system to store data" | "I want to save my preferences" |
| "So that the database is updated" | "So that my settings persist across sessions" |

#### 2c. Write Acceptance Criteria

Start from `feature.acceptanceCriteriaDraft`. For each draft item:
- Rewrite as a specific, testable condition in present tense
- Each AC = one verifiable assertion (not compound "and" statements)
- Add context: "When X, then Y" or "Given Z, <system> shows/returns/prevents..."

Expand draft ACs to cover:
1. Happy path (primary success scenario)
2. At least one validation rule
3. At least one error/edge condition (if inferable from source)

Format: `**AC-NN**: <condition>`

Minimum 3 ACs per story. Maximum 10.

#### 2d. Populate Technical Context

From `req-analysis.json#nonFunctionalRequirements` + `feature.description`:

- `Has UI`: yes if feature involves screens, forms, or user interactions
- `Has API`: yes if feature mentions endpoints, requests, or service calls
- `External Systems`: any third-party services, databases, or integrations mentioned
- `Performance SLAs`: pull relevant NFRs (e.g., "response < 2s")
- `Security / Compliance`: pull relevant NFRs (e.g., "HIPAA", "requires authentication")
- Mark fields as `N/A` if not applicable, never leave blank

#### 2e. Write Notes Section

From `feature.openQuestions` and `feature.description`:
- Open Questions: items that need stakeholder clarification before development begins
- Constraints: technical, regulatory, or business constraints mentioned in source
- Out of Scope: any explicit exclusions mentioned in source for this feature

If no open questions: omit that subsection entirely.

### Step 3: Apply Quality Rules

Read and enforce `rules/user-story-quality.md` (resolve via PLUGIN_DIR) before saving each file.

Reject and rewrite if:
- `So that` describes a technical outcome, not user value
- Any AC is vague ("works correctly", "is displayed", "functions as expected")
- ACs contain compound assertions ("and")
- Story has < 3 ACs

### Step 4: Resolve Output Mode

```
auto:
  if sourceFiles.count == 1  → mode = "grouped-single"   (one output file for the whole source)
  if sourceFiles.count > 1   → mode = "grouped-by-source" (one output file per source file)

single:
  → mode = "combined"   (always one file: US-all-<projectName>.md)

separate:
  → mode = "separate"   (always one file per story: US-NNN-<slug>.md)
```

### Step 5: Save Files

#### Mode: `grouped-single` or `grouped-by-source`

Write stories as H2 sections inside a grouped file. Each group file covers one source document.

**File name**: `US-<source-slug>.md`

#### Mode: `combined`

Write all stories from all sources into a single file: `US-all-<projectName>.md`

#### Mode: `separate`

Write each story to its own file: `US-<NNN>-<feature-slug>.md`

### Step 6: Write US-INDEX.md

Always written regardless of output mode.

```markdown
# User Stories Index

| ID | Title | Actor | ACs | File | Source Section |
|----|-------|-------|-----|------|----------------|
| US-001 | Patient Login with MFA | Patient | 5 | [US-prd-v2.md#us-001](US-prd-v2.md#us-001) | §3.2 |
```

### Step 7: Report

```
✅ User Stories Generated

Stories: N files written to <outputBase>
Index:   <outputBase>/US-INDEX.md

💡 Next Steps:
  /sqe-kit:gen-manual-tests --story <outputBase>/US-001-<slug>.md
```

---

## Notes

- Each story file is self-contained and can be passed directly to `/sqe-kit:gen-manual-tests` or `/sqe-kit:e2e`
- `openQuestions` from the source are preserved verbatim — do not try to answer them
- If a feature has very sparse detail in the source, mark ACs as `[TO BE CONFIRMED]` rather than inventing requirements
