# Intake Agent

**Single Responsibility**: Normalize, validate, and classify all user-provided inputs. Always
runs as Step 0 in every generation pipeline. Produces `intake.summary.json` — the single
source of truth consumed by all downstream agents.

---

## Inputs (passed by calling command)

| Parameter | Type | Description |
|---|---|---|
| `appUrl` | string \| null | Application URL |
| `domFiles` | array \| string \| null | One or more DOM snapshot files. Accepts: (a) array of `{ pageName, file }` objects, (b) a single file path string (backward-compatible), or (c) null |
| `testCasesRaw` | string \| null | Raw test case text or file path |
| `kitConfig` | object | Full kit/kit.config.json contents (resolve via PLUGIN_DIR) |

### domFiles input formats

```
// Multi-page (preferred for cross-page flows):
[
  { "pageName": "HomePage",          "file": "inputs/snapshots/amazonHome-page.html" },
  { "pageName": "SearchResultsPage", "file": "inputs/snapshots/amazonSearch-page.html" }
]

// Single file (legacy / backward-compatible):
"inputs/snapshots/amazonHome-page.html"

// Also accepted: comma-separated string of paths (auto-assigns pageNames from filename):
"inputs/snapshots/amazonHome-page.html, inputs/snapshots/amazonSearch-page.html"
```

---

## Step 0: Brownfield Detection

Before processing inputs, check if `.sqe/project-fingerprint.json` exists in the current
working directory:

```
if .sqe/project-fingerprint.json exists:
  read fingerprint → mode = "brownfield"
  outputBase = CWD (the existing project root)
  store fingerprint for downstream agents:
    - fingerprint.dirs.pom       → outputDirs.pom
    - fingerprint.dirs.nonbdd    → outputDirs.nonbdd  (or null if nonbdd not applicable)
    - fingerprint.dirs.bdd       → outputDirs.bdd      (or null if bdd not applicable)
    - fingerprint.dirs.steps     → outputDirs.steps    (or null)
    - fingerprint.dirs.fixtures  → outputDirs.fixtures  (or null)
    - fingerprint.testStyle      → use as testStyle (overrides kit default)
    - fingerprint.importAlias    → forwarded to pom-agent and testgen-agent
    - fingerprint.constructorPattern → forwarded to pom-agent
    - fingerprint.locatorStyle   → forwarded to locator-agent
    - fingerprint.existingPOMs   → forwarded to pom-agent (conflict check)
    - fingerprint.existingTests  → forwarded to testgen-agent and bdd-agent (conflict check)
    - fingerprint.existingFeatures → forwarded to bdd-agent (dedup)
  log: "Brownfield mode: existing project detected at <CWD>"

else:
  mode = "greenfield"
  outputBase = outputs/<projectName>/  (from kitConfig.outputDir)
  log: "Greenfield mode: writing to outputs/<projectName>/"
```

---

## Step 1: Validate and classify inputs

```
appUrl is present AND starts with "http"  → appUrl = <url>; else appUrl = null

domFiles normalization:
  - null or empty              → domFiles = []
  - single string path         → domFiles = [{ pageName: inferPageName(path), file: path }]
  - comma-separated string     → split, map each to { pageName: inferPageName(path), file: path }
  - array of { pageName, file } objects → validate each entry:
      - read file → confirm HTML content
      - if file not found or not HTML → remove entry, add warning
  - result: domFiles = validated array (may be empty)

inferPageName(filePath):
  - strip directory and extension from filename
  - convert to PascalCase
  - e.g. "inputs/snapshots/amazonHome-page.html" → "AmazonHomePage"
  - e.g. "inputs/snapshots/amazonSearch-page.html" → "AmazonSearchPage"
  - if pageName is explicitly provided by caller, always use that value

testCasesRaw is present:
  - if it looks like a file path (ends with .md, .txt, .csv, .json) → read the file first
  - else treat as inline text
```

**Warnings to collect:**
- No appUrl AND domFiles is empty → add warning: `"No locator source provided. Strategy will be AI-guided. Locators will be inferred and MUST be verified against the real application."`
- No testCasesRaw → add warning: `"No test cases provided. Generation will produce scaffold stubs only."`
- appUrl present but not reachable → add warning: `"Could not reach URL during intake. Proceeding with strategy=playwright-mcp but locator extraction may fail."`
- domFiles entry whose file could not be read → add warning per file: `"DOM file '<path>' could not be read and was skipped."`
- testCases reference pages not covered by any domFiles entry → add warning: `"Test case <id> references page '<page>' but no DOM snapshot was provided for it. Locators for that page will be AI-guided."`

---

## Step 2: Determine strategy and testStyle

```
if appUrl != null          → strategy = "playwright-mcp"
elif domFiles.length > 0   → strategy = "dom-based"
else                       → strategy = "ai-guided"
```

After determining strategy, resolve `testStyle` and `outputDirs`:

```
Brownfield mode:
  testStyle = fingerprint.testStyle  ← always use fingerprint value
  outputDirs = {
    "pom":      fingerprint.dirs.pom,
    "nonbdd":   fingerprint.dirs.nonbdd ?? null,
    "bdd":      fingerprint.dirs.bdd ?? null,
    "steps":    fingerprint.dirs.steps ?? null,
    "fixtures": fingerprint.dirs.fixtures ?? null
  }

Greenfield mode:
  if --bdd flag was passed by calling command:
    testStyle = "bdd"
  else:
    testStyle = kitConfig.strategyDefaults[strategy].testStyle
                ?? kitConfig.testStyle   ← fallback to top-level default

  outputDirs = {
    "pom":      kitConfig.namingConventions.pomDir,              // always "src/pages"
    "nonbdd":   kitConfig.namingConventions.testDir,             // "tests/nonbdd"
    "bdd":      kitConfig.namingConventions.featureFileDir,      // "tests/bdd"
    "steps":    kitConfig.namingConventions.stepsFileDir,        // "tests/bdd"
    "fixtures": kitConfig.namingConventions.fixturesDir          // "src/fixtures"
  }
```

These are written verbatim into `intake.summary.json` so all downstream agents use them
without re-reading `kit/kit.config.json`.

---

## Step 3: Parse test cases

Use `skills/test-case-parser.md` (resolve via PLUGIN_DIR) to parse `testCasesRaw` into structured objects.

Supported input formats: numbered list, plain English, partial Gherkin, table.

If `testCasesRaw` is null: set `testCases = []`, add warning.

---

## Step 4: Write `intake.summary.json`

Write to `<outputBase>/intake.summary.json` (greenfield: `outputs/<projectName>/intake.summary.json`;
brownfield: `<CWD>/.sqe/intake.summary.json`):

```json
{
  "mode": "greenfield | brownfield",
  "appUrl": null,
  "domFiles": [
    { "pageName": "AmazonHomePage",   "file": "inputs/snapshots/amazonHome-page.html" },
    { "pageName": "AmazonSearchPage", "file": "inputs/snapshots/amazonSearch-page.html" }
  ],
  "strategy": "dom-based",
  "testStyle": "nonbdd",
  "outputDirs": {
    "pom":      "src/pages",
    "nonbdd":   "tests/nonbdd",
    "bdd":      "tests/bdd",
    "steps":    "tests/bdd",
    "fixtures": "src/fixtures"
  },
  "brownfieldContext": {
    "importAlias": "@pages",
    "constructorPattern": "private-readonly",
    "locatorStyle": "testid",
    "existingPOMs": ["src/pages/LoginPage.ts"],
    "existingTests": ["tests/nonbdd/TC-001-login.spec.ts"],
    "existingFeatures": ["tests/bdd/login.feature"]
  },
  "testCases": [
    {
      "id": "TC-001",
      "title": "Verify user can search for a product",
      "tags": ["smoke"],
      "preconditions": ["User is on the Amazon homepage"],
      "steps": [
        { "order": 1, "action": "navigate", "target": "amazon homepage",  "value": "https://www.amazon.in", "pageContext": "AmazonHomePage" },
        { "order": 2, "action": "assert",   "target": "search bar",       "value": null,                    "pageContext": "AmazonHomePage" },
        { "order": 3, "action": "fill",     "target": "search bar",       "value": "iPhone",                "pageContext": "AmazonHomePage" },
        { "order": 4, "action": "click",    "target": "search icon",      "value": null,                    "pageContext": "AmazonHomePage",   "transitionTo": "AmazonSearchPage" },
        { "order": 5, "action": "assert",   "target": "search results",   "value": null,                    "pageContext": "AmazonSearchPage" },
        { "order": 6, "action": "assert",   "target": "product listings", "value": "iPhone",                "pageContext": "AmazonSearchPage" }
      ],
      "expectedResults": [
        "Search results page is displayed",
        "Products related to iPhone appear in the listing"
      ],
      "pages": ["AmazonHomePage", "AmazonSearchPage"],
      "pageTransitions": [
        { "atStep": 4, "from": "AmazonHomePage", "to": "AmazonSearchPage" }
      ]
    }
  ],
  "warnings": [],
  "generatedAt": "2026-03-06T00:00:00.000Z",
  "kitId": "kit-u1"
}
```

The `brownfieldContext` field is `null` in greenfield mode.

---

## Step 5: Surface warnings

If `warnings` array is non-empty:
- Print each warning prominently (with a ⚠️ prefix)
- For `ai-guided` strategy: pause and ask user to confirm before downstream agents proceed
- For other warnings: proceed but keep warnings visible in the summary

---

## Rules

- This agent NEVER modifies any existing test code or configuration
- It reads files but only writes `intake.summary.json`
- If `intake.summary.json` already exists: compare checksums; if inputs are unchanged, skip re-parsing and re-use existing file (with a note: "Using cached intake.summary.json")
- Page inference: use `skills/test-case-parser.md` skill's page inference rules to populate `pages`, `step.pageContext`, `step.transitionTo`, and `pageTransitions` in each test case
- Page coverage check: after parsing test cases, cross-reference every `pageContext` value in steps against `domFiles[].pageName`. Any unmatched page name → add warning and mark that page as `"strategy": "ai-guided"` in the `domFiles` entry
- `domFile` (singular, legacy) is accepted as a backward-compatible alias for `domFiles` with a single entry
