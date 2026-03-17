# Reporting Agent

**Single Responsibility**: Install, configure, and verify test reporting integrations for the
active kit. Idempotent — safe to run on already-configured projects.

---

## Inputs (passed by calling command)

| Parameter | Type | Description |
|---|---|---|
| `reporters` | string[] | e.g., `["playwright-html", "allure"]` |
| `kitConfig` | object | Full `kit/kit.config.json` (resolve via PLUGIN_DIR) |
| `projectPath` | string | Path to the generated project root |
| `reset` | boolean | If true, remove existing reporter config and start fresh |

---

## Supported Reporter Matrix

| Reporter ID | Kit Support | Package | Config Location |
|---|---|---|---|
| `playwright-html` | U1, U2, H1 | built-in `@playwright/test` | `playwright.config.ts` |
| `allure` (Playwright) | U1, U2, H1 | `allure-playwright` | `playwright.config.ts` + `allure.config.ts` |
| `allure` (TestNG) | U3, U4, A1, H2 | `allure-testng` | `pom.xml` + `allure.properties` |
| `extent` | U3, U4, H2 | `extentreports` | `ExtentReports` config class |
| `allure` (pytest) | A2, H3 | `allure-pytest` | `pytest.ini` + `conftest.py` |
| `pytest-html` | A2, H3 | `pytest-html` | `pytest.ini` addopts |

---

## Configuration Steps by Reporter

### `playwright-html` (Kit-U1/U2/H1)

1. Check if `playwright.config.ts` has `'html'` in the `reporter` array
2. If missing or `reset = true`:
   - Add `['html', { open: 'never' }]` to the reporter array
   - Ensure `outputFolder: 'playwright-report'` is set
3. Add npm script if missing: `"report": "npx playwright show-report"`

### `allure` for Playwright (Kit-U1/U2/H1)

1. Check `package.json` devDependencies for `allure-playwright`
2. If missing: `npm install --save-dev allure-playwright@^3.0.0`
3. In `playwright.config.ts` reporter array, add:
   ```typescript
   ['allure-playwright', {
     detail: true,
     outputFolder: 'allure-results',
     suiteTitle: false
   }]
   ```
4. Render `allure.config.ts` from `templates/allure.config.ts.tmpl` (resolve via PLUGIN_DIR) if it doesn't exist
5. Add npm scripts:
   ```json
   "test:allure": "playwright test && npx allure generate allure-results --clean -o allure-report",
   "allure:open": "npx allure open allure-report"
   ```
6. Verify: run `npx playwright test --reporter=allure-playwright -- --list` (dry-run, checks reporter loads)

### `allure` for TestNG/Maven (Kit-U3/U4/A1/H2)

1. In `pom.xml`, add to `<dependencies>`:
   ```xml
   <dependency>
     <groupId>io.qameta.allure</groupId>
     <artifactId>allure-testng</artifactId>
     <version>2.25.0</version>
   </dependency>
   ```
2. Add `aspectj` dependency and `maven-surefire-plugin` configuration for AspectJ weaving
3. Create `src/test/resources/allure.properties`:
   ```properties
   allure.results.directory=target/allure-results
   ```

### `extent` (Kit-U3/U4/H2)

1. Add ExtentReports dependency to `pom.xml`
2. Create `src/test/java/com/<project>/config/ExtentReportConfig.java`
3. Add TestNG listener in `testng.xml`

### `allure` for pytest (Kit-A2/H3)

1. Add `allure-pytest` to `requirements.txt` or `pyproject.toml`
2. In `pytest.ini`:
   ```ini
   [pytest]
   addopts = --alluredir=allure-results
   ```
3. Add decorators reminder in generated test files

### `pytest-html` (Kit-A2/H3)

1. Add `pytest-html` to dependencies
2. In `pytest.ini`:
   ```ini
   addopts = --html=reports/report.html --self-contained-html
   ```

---

## Verification

After configuring reporters:
1. Run a minimal dry-run: `npx playwright test --reporter=list --project=chromium -- --list`
2. Confirm reporter packages load without error
3. Confirm output directories exist or will be created on first test run

Report: "Reporting configured. Run `npm run test:allure` to execute tests with full reporting."
