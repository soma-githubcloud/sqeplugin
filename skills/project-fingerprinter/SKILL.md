# Skill: Project Fingerprinter

Apply this skill to scan an existing Playwright TypeScript automation project and extract its
conventions into a `project-fingerprint.json` file. Used exclusively by `agents/project-scanner-agent.md`.

---

## Purpose

The fingerprint captures the project's existing conventions so that all subsequent SQE generation
commands produce artifacts that match the project's style — same import aliases, same constructor
patterns, same directory structure.

---

## Scan Steps

### Step 1: Discover Project Root

The current working directory is the project root. Confirm by checking for:
- `package.json` — required
- `playwright.config.ts` or `playwright.config.js` — required for Playwright projects
- `tsconfig.json` — optional but expected for TypeScript projects

If `package.json` is missing: abort with error "No package.json found. Is this a Node.js project?"

### Step 2: Read package.json

Extract:
- `name` → `packageJson.name`
- `version` → `packageJson.version`
- Check `dependencies` + `devDependencies` for:
  - `@playwright/test` → confirms Playwright project
  - `@cucumber/cucumber` → BDD project
  - `allure-playwright` → Allure reporting

### Step 3: Detect Test Style

```
If @cucumber/cucumber in dependencies → testStyle = "bdd" (or "both" if specs also exist)
Else if *.spec.ts files exist → testStyle = "nonbdd"
Else → testStyle = "both" (default)
```

### Step 4: Discover Directories

Search for existing file patterns to detect project directory structure:

```
POM files:     glob **/*Page.ts, **/*PO.ts, **/*page.ts (exclude node_modules)
Spec files:    glob **/*.spec.ts (exclude node_modules)
Feature files: glob **/*.feature (exclude node_modules)
Step files:    glob **/*.steps.ts (exclude node_modules)
Fixture files: glob **/*.fixtures.ts (exclude node_modules)
```

From the discovered files, extract directory paths:
- `dirs.pom` = common directory of POM files (e.g., `src/pages`, `tests/pages`, `pageObjects`)
- `dirs.nonbdd` = common directory of spec files (e.g., `tests`, `tests/e2e`, `src/test`)
- `dirs.bdd` = common directory of feature files (e.g., `features`, `tests/bdd`, `e2e/features`)
- `dirs.steps` = common directory of step files (e.g., `step-definitions`, `steps`)
- `dirs.fixtures` = common directory of fixture files

**Directory detection algorithm**: For each category, take all discovered file paths, extract their
parent directory, find the most common parent prefix, use that as the directory.

### Step 5: Detect Import Alias

Scan `tsconfig.json` for `compilerOptions.paths`:
```json
{
  "paths": {
    "@pages/*": ["src/pages/*"],
    "@fixtures/*": ["src/fixtures/*"]
  }
}
```

If `@pages/*` or similar found → `importAlias = "@pages"`
If no alias → `importAlias = null` (relative imports will be used)

Also check `playwright.config.ts` for `use.baseURL` → `baseUrl`.

### Step 6: Detect Constructor Pattern

Read 2-3 existing POM files and detect constructor style:

Pattern A — `private readonly` (TypeScript strict):
```typescript
constructor(private readonly page: Page) {}
```
→ `constructorPattern = "private-readonly"`

Pattern B — standard assignment:
```typescript
private page: Page;
constructor(page: Page) { this.page = page; }
```
→ `constructorPattern = "standard"`

Default if none found: `constructorPattern = "private-readonly"` (kit-u1 standard)

### Step 7: Detect Locator Style

Sample 2-3 existing POM files and count locator method usage:
- If majority use `getByTestId` → `locatorStyle = "testid"`
- If majority use `getByRole` → `locatorStyle = "role"`
- If majority use `locator()` with CSS → `locatorStyle = "css"`
- Mixed → `locatorStyle = "mixed"`

### Step 8: Detect Naming Conventions

From discovered files, extract:
- `pageObjectSuffix`: "Page" (from `LoginPage.ts`) or "PO" (from `LoginPO.ts`)
- `specFileSuffix`: ".spec.ts" or ".test.ts"
- `stepFileSuffix`: ".steps.ts" or ".step.ts"

### Step 9: Catalog Existing Artifacts

```
existingPOMs     = list of discovered POM file paths (relative to project root)
existingTests    = list of discovered spec file paths (relative to project root)
existingFeatures = list of discovered feature file paths (relative to project root)
existingSteps    = list of discovered step definition file paths (relative to project root)
```

These lists are used by generation commands to check for conflicts before writing new files.

---

## Output: project-fingerprint.json

Write to `.sqe/project-fingerprint.json` (create `.sqe/` directory if it doesn't exist):

```json
{
  "version": "1.0",
  "scannedAt": "2026-03-17T10:00:00Z",
  "packageJson": {
    "name": "my-app-e2e",
    "version": "1.0.0"
  },
  "testStyle": "nonbdd",
  "dirs": {
    "pom": "src/pages",
    "nonbdd": "tests/e2e",
    "bdd": "features",
    "steps": "step-definitions",
    "fixtures": "src/fixtures"
  },
  "namingConventions": {
    "pageObjectSuffix": "Page",
    "specFileSuffix": ".spec.ts",
    "stepFileSuffix": ".steps.ts"
  },
  "importAlias": "@pages",
  "constructorPattern": "private-readonly",
  "locatorStyle": "testid",
  "baseUrl": "http://localhost:3000",
  "existingPOMs": [
    "src/pages/LoginPage.ts",
    "src/pages/DashboardPage.ts"
  ],
  "existingTests": [
    "tests/e2e/login.spec.ts"
  ],
  "existingFeatures": [],
  "existingSteps": []
}
```

---

## Validation

After writing the fingerprint, confirm to the user:

```
✅ Project scanned successfully

Project : my-app-e2e
Style   : nonbdd
POMs    : src/pages/ (2 files found)
Tests   : tests/e2e/ (1 file found)
Alias   : @pages
Pattern : private-readonly

Fingerprint written to: .sqe/project-fingerprint.json

Run /sqe-kit:generate-automation-tests --cases "..." --url <url> to generate artifacts.
```

---

## Re-scan Behavior

If `.sqe/project-fingerprint.json` already exists:
- Report: "Existing fingerprint found (scanned: <date>). Re-scanning..."
- Overwrite with new scan results
- Show diff summary: "Changed: dirs.nonbdd (was tests/, now tests/e2e/)"
