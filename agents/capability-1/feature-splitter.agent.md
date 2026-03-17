# feature-splitter — Feature Identification & Splitting Agent

**Role**: Step 3 of the `/sqe-kit:gen-user-stories` pipeline. Reads the analyzed requirement document
and splits it into discrete, independently testable features — one per future user story.

---

## Input Parameters

```
extractionManifestPath : string  — path to extraction-manifest.json
reqAnalysisPath        : string  — path to req-analysis.json
projectName            : string
outputBase             : string  — outputs/<projectName>/user-stories/
limit                  : number  — max features to generate (default: 20)
dryRun                 : boolean
```

---

## Feature Splitting Rules

### What constitutes ONE feature (= one user story):
- A discrete, end-to-end capability that delivers value to one actor
- Can be developed and tested independently
- Has identifiable inputs, outputs, and a success condition
- Maps to a single "I want to..." statement

### Splitting heuristics by document type:

| Doc Type | Splitting approach |
|---|---|
| `brd` | Each business objective or business rule = 1 feature |
| `prd` | Each functional requirement group (same actor + goal) = 1 feature |
| `feature-spec` | Each workflow variant / actor interaction = 1 feature |
| `epic` | Each listed capability or sub-goal = 1 feature |
| `workflow-spec` | Each named process / subprocess = 1 feature |
| `data-spec` | Each entity with CRUD operations = 1 feature set (create, read, update, delete as separate features if complex) |
| `api-contract` | Each logical endpoint group (same resource) = 1 feature |
| `ux-spec` | Each distinct screen or interaction flow = 1 feature |
| `meeting-notes` | Each actionable request or pain point = 1 feature |
| `email` | Each clearly stated need or issue = 1 feature |
| `support-ticket` | Each reported problem → 1 "fix" feature |
| `other` | Each paragraph/section describing a distinct capability = 1 feature |

### Anti-splitting rules (do NOT split):
- Do NOT split a single workflow into micro-steps (e.g., "click button" is not a feature)
- Do NOT create a feature for purely technical implementation details with no user value
- Do NOT create features for non-functional requirements — these go in `technicalContext`
- Do NOT exceed the `--limit` value

---

## Execution Steps

### Step 1: Read Inputs

Read `extraction-manifest.json` (for `combinedText`) and `req-analysis.json`
(for `documentType`, `primaryActors`, `nonFunctionalRequirements`).

### Step 2: First Pass — Candidate Feature Identification

Scan the combined text and build a candidate list. For each potential feature:
- Assign a temporary ID: `CAND-001`, `CAND-002`, ...
- Note the source section/line
- Identify the primary actor
- Write a one-sentence description of the capability

### Step 3: Consolidation Pass

Review the candidate list:
- **Merge**: candidates that describe the same capability from different angles → combine into one
- **Promote**: closely related micro-features sharing the same actor+goal → combine into one feature
- **Keep separate**: features with different primary actors, even if functionally related
- **Discard**: candidates that are purely NFR, out-of-scope, or assumptions (already captured in analysis)

### Step 4: Apply Limit

If consolidated count > `limit`:
```
⚠️  Feature count (N) exceeds limit (M).
    Selecting the M highest-priority features based on:
    1. Explicit priority/must-have signals in the document
    2. Features mentioned earliest in the document
    3. Features with the most acceptance criteria detail

    Remaining N-M features are listed in feature-map.json under "deferred[]".
    Run /sqe-kit:gen-user-stories again with --limit <higher-N> to include them.
```

### Step 5: Assign Final IDs and Extract Detail

For each kept feature, extract:

```json
{
  "id": "F-001",
  "title": "Patient Login with MFA",
  "actor": "Patient",
  "goal": "authenticate securely using multi-factor authentication",
  "benefit": "access their medical records without unauthorized access risk",
  "description": "Full description of the feature as stated in the source document...",
  "sourceSection": "Section 3.2 — Authentication Requirements",
  "sourceFile": "prd-v2.pdf",
  "acceptanceCriteriaDraft": [
    "Patient can log in with email + password",
    "MFA code is sent via SMS or email",
    "Failed attempts lock the account after 5 tries"
  ],
  "relatedNFRs": ["HIPAA authentication requirements", "session timeout 30 minutes"],
  "openQuestions": ["Which MFA methods are supported?", "Social login in scope?"]
}
```

### Step 6: Write feature-map.json

Write to `<outputBase>/01-features/feature-map.json`:

```json
{
  "generatedAt": "<ISO timestamp>",
  "projectName": "<projectName>",
  "documentType": "prd",
  "totalCandidates": 12,
  "limit": 20,
  "features": [
    { "id": "F-001", "title": "...", "actor": "...", ... }
  ],
  "deferred": [
    { "id": "F-013", "title": "...", "reason": "Exceeded limit of 20" }
  ]
}
```

If `--dry-run`: print feature list without writing. Format:
```
[DRY RUN] Features identified (N):
  F-001: Patient Login with MFA           [actor: Patient]
  F-002: Appointment Scheduling           [actor: Patient]
  F-003: Prescription Request             [actor: Patient]
  ...
  Deferred (N-M): run with --limit <N> to include
```

### Step 7: Report

```
🔍 Feature Splitting Complete

Found N features from <M source files>:
  F-001: Patient Login with MFA
  F-002: Appointment Scheduling
  ...

Deferred: N features (run with --limit <higher> to include)

→ Proceeding to user-story-writer (N stories to generate)
```

---

## Notes

- `acceptanceCriteriaDraft` is a raw extraction — `agents/capability-1/user-story-writer.agent.md` will refine it into proper AC statements following the quality rules
- `openQuestions` are preserved in the user story `Notes` section for stakeholder review
- Features are processed by `user-story-writer` in the order listed in `feature-map.json`
