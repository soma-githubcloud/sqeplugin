# Scaffolder Agent

**Single Responsibility**: Create a production-ready project directory structure with all
configuration files, install dependencies, and verify the scaffold compiles cleanly.

---

## Inputs (passed by calling command)

| Parameter | Type | Description |
|---|---|---|
| `kitConfig` | object | Full `kit/kit.config.json` contents (resolve via PLUGIN_DIR) |
| `projectPath` | string | Absolute or relative path where the project is created |
| `testStyle` | string | `"non-bdd"` \| `"bdd"` |
| `projectName` | string | Used in package.json name and directory naming |

---

## Greenfield Scaffold (mode = "greenfield")

### Step 1: Create directories

```
<projectPath>/
├── pages/        ← Page Object classes
├── tests/        ← Spec files (non-BDD)
├── features/     ← .feature files (BDD only)
├── steps/        ← Step definitions (BDD only)
├── fixtures/     ← Test data and Playwright fixtures
└── allure-results/  ← (auto-created by runner; pre-create for .gitkeep)
```

Only create `features/` and `steps/` if `testStyle = "bdd"`.

### Step 2: Render and write configuration files

Load templates from `templates/` (resolve via PLUGIN_DIR):

**`package.json`** from `templates/package.json.tmpl`:
- Name: `<projectName>-tests`
- Scripts:
  ```json
  {
    "test":         "playwright test",
    "test:headed":  "playwright test --headed",
    "test:allure":  "playwright test && npx allure generate allure-results --clean -o allure-report",
    "allure:open":  "npx allure open allure-report",
    "lint":         "tsc --noEmit"
  }
  ```
- devDependencies (PINNED versions — do not use `latest`):
  ```json
  {
    "@playwright/test": "^1.44.0",
    "allure-playwright": "^3.0.0",
    "typescript": "^5.4.0"
  }
  ```
  If BDD: also add `"@cucumber/cucumber": "^10.0.0"`, `"@cucumber/playwright": "^0.0.1"`

**`playwright.config.ts`** from `templates/playwright.config.ts.tmpl`:
- `testDir`: `./tests` (or `./features` for BDD)
- `baseURL`: `process.env.BASE_URL || 'http://localhost:3000'` (user sets via env var)
- `reporter`: `[['html'], ['allure-playwright']]`
- `use`: `{ screenshot: 'only-on-failure', video: 'retain-on-failure', trace: 'on-first-retry' }`
- `projects`: single Chromium project for default setup

**`tsconfig.json`** (standard, not from template):
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "strict": true,
    "esModuleInterop": true,
    "baseUrl": ".",
    "paths": { "@pages/*": ["pages/*"], "@fixtures/*": ["fixtures/*"] }
  },
  "include": ["**/*.ts"],
  "exclude": ["node_modules"]
}
```

**`.gitignore`**:
```
node_modules/
playwright-report/
allure-results/
allure-report/
test-results/
```

**`allure.config.ts`** from `templates/allure.config.ts.tmpl`:
- Basic configuration for suite metadata

### Step 3: Install dependencies

```bash
cd <projectPath>
npm install
npx playwright install chromium
```

### Step 4: Verify scaffold

```bash
npx tsc --noEmit
```

Must exit with code 0. If errors exist, fix them before reporting completion.

### Step 5: Report
- List all created files and directories
- Confirm `node_modules/` is installed
- Show first command to run: `/sqe-kit:generate-automation-tests`

---

## For Existing Framework (mode = "existing")

Used by Kit-U2, Kit-U4. Before scaffolding, read `agents/capability-3/framework-setup-agent.md`
(resolve via PLUGIN_DIR) and execute it first to discover the existing structure. The setup agent
returns a `frameworkMap` that this agent uses to decide:
- Where to write new POM files (don't duplicate existing ones)
- How to update existing `playwright.config.ts` or `pom.xml` rather than overwriting it
- Which dependencies are already installed vs. need to be added

---

## Java Scaffold (Kit-U3, U4, H2)

Uses Maven project structure. Key files rendered from templates:
- `pom.xml` — dependencies: selenium-java, testng/junit5, allure-testng, cucumber (if BDD)
- `src/test/java/com/<projectName>/` — package structure
- `testng.xml` or `junit-platform.properties` — runner config
- `src/test/resources/allure.properties` — report config

Verification: `mvn compile -q` (must succeed)

---

## Python Scaffold (Kit-A2, H3)

Key files:
- `pyproject.toml` or `requirements.txt` — dependencies: pytest, requests, allure-pytest
- `conftest.py` (root level) — base fixtures
- `pytest.ini` — `testpaths = tests`, `addopts = --alluredir=allure-results`
- `tests/`, `pages/` (H3 only)

Verification: `pytest --collect-only -q`
