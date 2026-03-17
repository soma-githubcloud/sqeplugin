# POM Agent

**Single Responsibility**: Generate Page Object Model TypeScript class files from
`selectors.json`. This agent does NOT generate test specs or feature files — POMs only.

---

## Inputs (passed by calling command)

| Parameter | Type | Description |
|---|---|---|
| `selectorsFile` | string | Path to `selectors.json` |
| `kitConfig` | object | Full kit/kit.config.json (resolve via PLUGIN_DIR) |
| `intakeSummary` | object | Contents of `intake.summary.json` (for brownfield context) |
| `overwrite` | boolean | Overwrite existing POM files (default: false) |
| `extend` | boolean | Append new locators to existing POM file without overwriting class |
| `dryRun` | boolean | Print planned files without writing |

---

## Step 0: Brownfield Conflict Check

If `intakeSummary.mode == "brownfield"`:
- Read `intakeSummary.brownfieldContext.existingPOMs` (array of existing POM file paths)
- For each `ComponentName` in `selectors.json`:
  - Derive expected POM filename (e.g., `LoginPage.ts`)
  - Check if any path in `existingPOMs` ends with that filename
  - If found AND `overwrite = false` AND `extend = false`:
    → Skip this component; log: "Skipped <ComponentName>.ts — already exists (use --extend or --overwrite)"
  - If found AND `extend = true`:
    → Read existing file; append new locators that don't already exist; preserve existing methods
  - If found AND `overwrite = true`:
    → Regenerate and overwrite

Extract brownfield template parameters:
- `constructorPattern` = `intakeSummary.brownfieldContext.constructorPattern`
  (e.g., `"private-readonly"` or `"standard"`)
- `importAlias` = `intakeSummary.brownfieldContext.importAlias`
  (e.g., `"@pages"` or `null`)

---

## Pre-generation checks

1. Read `selectorsFile` — validate structure: must have at least one component with at least one locator
2. If any locator has `score < 0.70` (from selector-scorer): warn the user about low-confidence selectors but proceed
3. If `overwrite = false`: list files in the POM output dir — skip components whose file already exists; report which are skipped
4. Read `skills/template-renderer.md` (resolve via PLUGIN_DIR) — apply rendering rules throughout

---

## Generation (TypeScript — Kit-U1)

For each `ComponentName` in `selectors.json` (not skipped):

1. Load template: `templates/page-object.ts.tmpl` (resolve via PLUGIN_DIR)

2. Build template context:
   ```json
   {
     "kitId": "kit-u1",
     "generatedDate": "2026-03-03",
     "className": "LoginPage",
     "pageRoute": "/login",
     "constructorPattern": "private-readonly",
     "importAlias": "@pages",
     "locators": [
       { "alias": "emailInput",   "method": "getByTestId", "selector": "email",   "score": 0.99 },
       { "alias": "submitButton", "method": "getByRole",   "selector": "button",  "score": 0.95, "name": "Sign in" }
     ],
     "actions": [
       { "methodName": "fillEmail",    "params": "email: string",   "body": "await this.emailInput.fill(email);" },
       { "methodName": "clickSubmit",  "params": "",                "body": "await this.submitButton.click();" },
       { "methodName": "login",        "params": "email: string, password: string",
         "body": "await this.fillEmail(email);\n    await this.fillPassword(password);\n    await this.clickSubmit();" }
     ]
   }
   ```

3. **Action method inference rules** (from locator aliases):
   - `*Input` or `*Field` → generate `fill<Alias>(value: string)` action
   - `*Button` or `*Btn` → generate `click<Alias>()` action
   - `*Select` or `*Dropdown` → generate `select<Alias>(option: string)` action
   - `*Checkbox` → generate `check<Alias>()` and `uncheck<Alias>()` actions
   - `*Link` → generate `click<Alias>()` action
   - If component has both email+password input → generate composite `login(email, password)` method

4. **pageRoute inference**: derive from component name (e.g., `LoginPage` → `/login`, `CheckoutPage` → `/checkout`)

5. **Output path**:
   - Brownfield: `<intakeSummary.outputDirs.pom>/<ComponentName>.ts`
   - Greenfield: `<outputDir>/<projectName>/src/pages/<ComponentName>.ts`

6. Render template → write to resolved output path

---

## Post-generation

1. Run `npx tsc --noEmit` (TypeScript)
2. Confirm: no raw selector strings in generated POM files
3. If `dryRun = true`: print planned file paths and method counts without writing anything

---

## Output summary format

```
POM Generation Complete
────────────────────────────────────────────
Created:  3 new files
Skipped:  1 (LoginPage.ts already exists)
Extended: 1 (DashboardPage.ts — 2 locators appended)
Warnings: 2 low-confidence selectors flagged

Files:
  ✓ src/pages/LoginPage.ts       (4 locators, 3 action methods)
  ✓ src/pages/DashboardPage.ts   (6 locators, 5 action methods)
  ✓ src/pages/CheckoutPage.ts    (8 locators, 6 action methods)
  ─ src/pages/ProfilePage.ts     (skipped — already exists)
```
