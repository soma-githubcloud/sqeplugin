# Project Scanner Agent

**Single Responsibility**: Scan an existing Playwright TypeScript project to detect its
conventions, directory structure, and code style — then write `.sqe/project-fingerprint.json`
to the project root. This enables brownfield mode for all downstream agents.

Invoked by `/sqe-kit:scan-project`.

---

## Input Parameters

```
projectRoot : string  — path to the existing project root (default: current working directory)
dryRun      : boolean — print what would be detected without writing fingerprint
```

---

## Execution Steps

### Step 1: Apply Project Fingerprinter Skill

Read and apply `skills/project-fingerprinter/SKILL.md` (resolve via PLUGIN_DIR) to scan
the project at `projectRoot`.

The skill returns a structured fingerprint object (see schema below).

### Step 2: Detect Project Type

Verify this is a Playwright TypeScript project:
- `package.json` must exist and contain `@playwright/test` in devDependencies
- `playwright.config.ts` or `playwright.config.js` must exist

If neither is found:
```
⚠️ No Playwright configuration found at <projectRoot>.
   This tool is designed for Playwright TypeScript projects.
   Continue anyway? [y/N]
```

### Step 3: Write Fingerprint

Unless `dryRun = true`:
- Create `.sqe/` directory if it doesn't exist
- Write `.sqe/project-fingerprint.json` with the full fingerprint object

### Step 4: Report

```
✅ Project Scan Complete

Project: <packageJson.name>
Type:    Playwright TypeScript
Style:   <testStyle>

Detected structure:
  POMs:      <dirs.pom>
  Tests:     <dirs.nonbdd>  (or N/A)
  Features:  <dirs.bdd>     (or N/A)
  Fixtures:  <dirs.fixtures>

Existing artifacts:
  <N> POMs found:     <existingPOMs summary>
  <N> tests found:    <existingTests summary>
  <N> features found: <existingFeatures summary>

Code conventions:
  Constructor pattern: <constructorPattern>
  Import alias:        <importAlias>
  Locator style:       <locatorStyle>

Fingerprint written: .sqe/project-fingerprint.json

💡 SQE Kit is now in brownfield mode for this project.
   Run /sqe-kit:generate-automation-tests to generate artifacts
   that integrate with your existing codebase.
```

---

## Fingerprint Schema

```json
{
  "generatedAt": "<ISO timestamp>",
  "schemaVersion": "1.0",
  "packageJson": {
    "name": "<project name from package.json>",
    "version": "<version>"
  },
  "testStyle": "nonbdd | bdd | both",
  "dirs": {
    "pom": "src/pages",
    "nonbdd": "tests/nonbdd",
    "bdd": "tests/bdd",
    "steps": "tests/bdd",
    "fixtures": "src/fixtures"
  },
  "existingPOMs": [
    "src/pages/LoginPage.ts",
    "src/pages/DashboardPage.ts"
  ],
  "existingTests": [
    "tests/nonbdd/TC-001-login.spec.ts",
    "tests/bdd/login.steps.ts"
  ],
  "existingFeatures": [
    "tests/bdd/login.feature"
  ],
  "importAlias": "@pages",
  "constructorPattern": "private-readonly",
  "locatorStyle": "testid",
  "namingConventions": {
    "specSuffix": ".spec.ts",
    "stepSuffix": ".steps.ts",
    "pomSuffix": "Page.ts",
    "featureSuffix": ".feature"
  }
}
```

### Field descriptions

| Field | Detected from |
|---|---|
| `testStyle` | Presence of `.feature` files (bdd), `.spec.ts` files (nonbdd), or both |
| `dirs.pom` | Directory containing POM class files (TypeScript classes with `Page` suffix) |
| `dirs.nonbdd` | Directory containing `.spec.ts` files |
| `dirs.bdd` | Directory containing `.feature` files |
| `dirs.steps` | Directory containing `.steps.ts` files (may equal `dirs.bdd`) |
| `dirs.fixtures` | Directory containing fixture/factory files |
| `existingPOMs` | All `.ts` files in `dirs.pom` |
| `existingTests` | All `.spec.ts` and `.steps.ts` files |
| `existingFeatures` | All `.feature` files |
| `importAlias` | Path alias for POM imports from `tsconfig.json` `paths` (e.g., `@pages`) |
| `constructorPattern` | `"private-readonly"` if `constructor(private readonly page: Page)` found; else `"standard"` |
| `locatorStyle` | Dominant locator type found in POM files: `"testid"` \| `"aria"` \| `"css"` \| `"mixed"` |

---

## Error Handling

| Condition | Action |
|---|---|
| `package.json` not found | Error: "Not a Node.js project. Run from project root." |
| No `playwright.config.ts` | Warning: "Playwright config not found — continuing with inferred structure" |
| Empty project (no TS files) | Fingerprint written with empty `existingPOMs`, `existingTests`, `existingFeatures` |
| `.sqe/` already exists | Overwrite `project-fingerprint.json`; log: "Updated existing fingerprint" |
