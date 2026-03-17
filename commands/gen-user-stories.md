# /sqe-kit:gen-user-stories — Generate User Stories from Requirement Documents

**Shift-left entry point.** Accepts requirement documents in any format (BRD, PRD, Feature Spec,
Epic, Meeting Notes, emails, images, etc.) and generates structured user story files ready
for `/sqe-kit:gen-manual-tests` or `/sqe-kit:e2e`. This is the leftmost step in the full quality pipeline:

```
Requirement Doc → User Stories → Manual TCs → Automation Artifacts
```

To continue into manual TCs immediately: use `--then-manual-tests`.
To run the full pipeline (req doc → user stories → manual TCs → automation): use `/sqe-kit:e2e --req`.

## Arguments

| Argument | Description |
|----------|-------------|
| `--req <path>` | **Required.** Path to requirement file, comma-separated list of files, or directory containing requirement docs |
| `--project <name>` | Override project name (default: `sqe-project`) |
| `--type <doctype>` | Hint the document type: `brd`, `prd`, `feature`, `epic`, `workflow`, `data`, `api`, `ux`, `meeting`, `email`, `ticket`, `other` (auto-detected if omitted) |
| `--limit <n>` | Max user stories to generate from one document (default: 20) |
| `--output-mode auto\|single\|separate` | Output file strategy (default: `auto` — see below) |
| `--dry-run` | Preview extracted features and planned files without writing |
| `--then-manual-tests` | After generating stories, run `/sqe-kit:gen-manual-tests` on each |
| `--style bdd\|nonbdd\|both` | Forwarded to `--then-manual-tests` (default: `both`) |

### `--output-mode` — Output File Strategy

Controls how user stories are grouped into files:

| Mode | Behaviour | Best for |
|------|-----------|----------|
| `auto` *(default)* | Single file when one source → one combined `US-<source-slug>.md`; separate files when multiple sources → one file per source | Most cases — smart default |
| `single` | Always one combined file `US-all-<project>.md` regardless of source count | Large PRDs where you want one review document |
| `separate` | Always one `US-NNN-<slug>.md` per story regardless of source count | When each story is fed independently to `/sqe-kit:gen-manual-tests` |

**`auto` mode rules in detail:**

```
if sourceFiles count == 1:
  → single output file: US-<source-slug>.md
    All stories appear as H2 sections inside the file.
    Each story still has its own US-NNN ID, AC list, and Technical Context block.

if sourceFiles count > 1:
  → one output file per source: US-<source-slug>.md (groups stories from that source)
    Stories from different sources are never mixed in one file.
    US-INDEX.md links to each grouped file.
```

**Examples:**

```bash
# Default auto — 1 PDF → 1 file with all stories inside
/sqe-kit:gen-user-stories --req docs/prd.pdf

# Default auto — 3 source files → 3 grouped files
/sqe-kit:gen-user-stories --req "docs/brd.pdf,docs/api-contract.yaml,docs/ux-spec.md"

# Force single combined file even with multiple sources
/sqe-kit:gen-user-stories --req docs/requirements/ --output-mode single

# Force separate file per story
/sqe-kit:gen-user-stories --req docs/prd.pdf --output-mode separate
```

---

### `--req` Input Resolution

| Input form | How handled |
|---|---|
| Single file path | Process that file |
| Comma-separated paths | Process each file independently; combine analysis |
| Directory path | Discover all supported files in directory (sorted alphabetically) |
| Glob pattern (e.g., `docs/*.pdf`) | Expand glob and process matching files |

### Supported File Formats

| Format | Tier | Method |
|--------|------|--------|
| `.md`, `.txt`, `.eml` | 1 — Native | Read directly |
| `.pdf` | 1 — Native | Read with pages param (chunked for large PDFs) |
| `.yaml`, `.yml`, `.json` | 1 — Native | Read + parse structure |
| `.xml`, `.bpmn` | 1 — Native | Read + parse XML |
| `.csv` | 1 — Native | Read + parse rows |
| `.png`, `.jpg`, `.jpeg` | 1 — Native | Read (multimodal visual analysis) |
| `.docx` | 2 — Shell | `pandoc` conversion → markdown (fallback: `docx2txt`) |
| `.xlsx` | 2 — Shell | `pandas` → CSV text (fallback: save as CSV) |
| `.fig` | 3 — Unsupported | User prompted to export as PDF or PNG |

### Examples

```bash
# Single PRD (PDF)
/sqe-kit:gen-user-stories --req docs/requirements/prd-v2.pdf

# Feature spec markdown
/sqe-kit:gen-user-stories --req docs/features/checkout-feature.md --type feature

# Multiple files combined
/sqe-kit:gen-user-stories --req "docs/brd.pdf,docs/api-contract.yaml"

# Entire requirements directory
/sqe-kit:gen-user-stories --req docs/requirements/

# Word document (auto-converts with pandoc)
/sqe-kit:gen-user-stories --req docs/feature-spec.docx

# Whiteboard photo or UX mockup screenshot
/sqe-kit:gen-user-stories --req whiteboard-session.png --type meeting

# Preview without writing
/sqe-kit:gen-user-stories --req docs/prd.pdf --dry-run

# Generate stories + manual TCs in one command
/sqe-kit:gen-user-stories --req docs/prd.pdf --then-manual-tests --style both
```

---

## Execution Pipeline

### Step 0: Setup

- `projectName` = `--project` argument OR `"sqe-project"` (default)
- `outputDir`   = `./outputs` (fixed)
- `outputBase`  = `<outputDir>/<projectName>/user-stories/`
- If `--dry-run`: announce dry-run mode

### Step 1: Ingest Requirement Documents

Read and execute `agents/capability-1/req-ingestor.agent.md` (resolve via PLUGIN_DIR). Pass:
```
reqInput     : value of --req (path, comma list, or directory)
projectName  : resolved project name
outputBase   : <outputDir>/<projectName>/user-stories/
docTypeHint  : value of --type (or null)
dryRun       : --dry-run flag
```

Outputs: `<outputBase>/00-analysis/extraction-manifest.json`

For each file processed, log:
```
📄 Processing: <filename> (<extension>, Tier N — <method>)
```

For each file skipped (Tier 3):
```
⚠️  Skipping: <filename> — unsupported format. <user action instructions>
```

### Step 2: Analyze Requirement Document

Read and execute `agents/capability-1/req-analyzer.agent.md` (resolve via PLUGIN_DIR). Pass:
```
extractionManifestPath : <outputBase>/00-analysis/extraction-manifest.json
projectName            : resolved project name
outputBase             : <outputBase>
docTypeHint            : --type (or null)
```

Outputs: `<outputBase>/00-analysis/req-analysis.json`

### Step 3: Split into Features

Read and execute `agents/capability-1/feature-splitter.agent.md` (resolve via PLUGIN_DIR). Pass:
```
extractionManifestPath : <outputBase>/00-analysis/extraction-manifest.json
reqAnalysisPath        : <outputBase>/00-analysis/req-analysis.json
projectName            : resolved project name
outputBase             : <outputBase>
limit                  : --limit value (default: 20)
dryRun                 : --dry-run flag
```

Outputs: `<outputBase>/01-features/feature-map.json`

If `--dry-run`: print feature list and stop here.

### Step 4: Generate User Story Files

Read and execute `agents/capability-1/user-story-writer.agent.md` (resolve via PLUGIN_DIR). Pass:
```
featureMapPath  : <outputBase>/01-features/feature-map.json
reqAnalysisPath : <outputBase>/00-analysis/req-analysis.json
projectName     : resolved project name
outputBase      : <outputBase>
dryRun          : --dry-run flag
```

Outputs:
- `<outputBase>/US-NNN-<slug>.md` — one per feature
- `<outputBase>/US-INDEX.md` — summary index

### Step 5: Quality Lint

Apply `rules/user-story-quality.md` (resolve via PLUGIN_DIR) across all generated stories:
- Verify all stories have US ID, source traceability, min 3 ACs
- Flag any AC that is vague or compound
- Flag any `So that` clause that describes a technical outcome
- Report warnings inline; do not fail generation on warnings

### Step 6: Summary

```
✅ User Story Generation Complete

Source:   <filename(s)>
Type:     <document type>
Project:  <projectName>
Output:   outputs/<projectName>/user-stories/

📋 Generated Files:
  Analysis:      outputs/<projectName>/user-stories/00-analysis/  (req-analysis.json)
  Feature Map:   outputs/<projectName>/user-stories/01-features/  (feature-map.json, N features)
  User Stories:  outputs/<projectName>/user-stories/              (N US files)
  Index:         outputs/<projectName>/user-stories/US-INDEX.md

📊 Summary:
  Stories generated: N
  Total ACs:         M
  Open questions:    P (review before handing to development)
  Deferred features: Q (re-run with --limit <higher> to include)

⚠️  Quality Warnings:
  US-003: AC-02 may be vague — review expected result specificity
  US-007: "So that" clause describes technical outcome — consider rewriting

💡 Next Steps:
  1. Review US-INDEX.md and resolve open questions with stakeholders
  2. Generate manual test cases for a story:
     /sqe-kit:gen-manual-tests --story outputs/<projectName>/user-stories/US-001-<slug>.md
  3. Full pipeline for all stories (manual TCs + automation):
     /sqe-kit:e2e --req <same-req-file> [--url <url>]
  4. Manual TCs for all stories without automation:
     /sqe-kit:gen-user-stories --req <same-file> --then-manual-tests
```

### Step 7: Pipeline Continuation (optional)

**If `--then-manual-tests`**: for each generated `US-NNN-*.md` file (in order):
```
/sqe-kit:gen-manual-tests --story <US-file> --project <projectName> --style <--style value>
```
Report progress after each story completes.

Stories are processed sequentially, not in parallel, to avoid resource contention.

---

## Output Structure

```
outputs/<projectName>/
└── user-stories/
    ├── 00-analysis/
    │   ├── extraction-manifest.json   ← per-file extraction results (Tier, method, warnings)
    │   └── req-analysis.json          ← doc type, actors, domain, NFRs, estimated feature count
    ├── 01-features/
    │   └── feature-map.json           ← F-001..F-NNN feature definitions
    ├── US-001-<slug>.md               ← individual user story (ready for /sqe-kit:gen-manual-tests)
    ├── US-002-<slug>.md
    ├── ...
    └── US-INDEX.md                    ← navigation index with traceability table
```

---

## Important Rules

- **Never modify source documents** — all output is written to `outputs/<projectName>/user-stories/`
- **Tier 3 files are skipped, not failed** — generation continues with remaining files; user is informed
- **`--limit` is per-run, not per-file** — when multiple files are combined, total story count ≤ limit
- **Idempotent**: re-running with the same `--req` and `--project` overwrites previous output after confirmation
- **`--dry-run`** stops after Step 3 (feature list preview) — no user story files are written
- **For full pipeline** (req → stories → manual TCs → automation), use `/sqe-kit:e2e --req` instead
