# CI Agent

**Single Responsibility**: Generate CI/CD pipeline YAML files that run the automation suite,
publish Allure reports, and upload artifacts. Supports GitHub Actions, Azure DevOps, GitLab CI.

---

## Inputs (passed by calling command)

| Parameter | Type | Description |
|---|---|---|
| `provider` | string | `"github"` \| `"azdo"` \| `"gitlab"` |
| `env` | string | `"staging"` \| `"prod"` |
| `matrix` | boolean | Multi-browser matrix |
| `schedule` | boolean | Add nightly cron |
| `kitConfig` | object | Full kit/kit.config.json (resolve via PLUGIN_DIR) |
| `integrations` | object | Contents of `configs/integrations.yml` (resolve via PLUGIN_DIR) |

---

## Step 1: Load CI skill and template

Read `skills/ci-scaffolder.md` (resolve via PLUGIN_DIR) — apply patterns throughout.
Load template: `templates/ci-github.yml.tmpl` (resolve via PLUGIN_DIR) (or provider-specific equivalent).

---

## Step 2: Determine test command

| Kit tech stack | Test command |
|---|---|
| playwright (Kit-U1/U2/H1) | `npx playwright test --grep "@smoke" --project=${{ matrix.browser }}` |
| selenium + testng (Kit-U3/U4/H2) | `mvn test -Dgroups="smoke" -q` |
| pytest (Kit-A2/H3) | `pytest -m smoke --alluredir=allure-results` |
| rest-assured (Kit-A1) | `mvn test -Dgroups="smoke" -q` |

---

## Step 3: Build template context

```json
{
  "projectName": "my-playwright-project",
  "provider": "github",
  "browsers": ["chromium"],
  "matrix": false,
  "schedule": false,
  "testCommand": "npx playwright test --grep \"@smoke\"",
  "baseUrlSecret": "BASE_URL",
  "allureEnabled": true,
  "artifactPaths": ["allure-report", "playwright-report"],
  "nodeVersion": "20",
  "publishToAllureServer": false
}
```

If `matrix = true`: expand `browsers` to `["chromium", "firefox", "webkit"]`
If `schedule = true`: add `cron: '0 2 * * *'` to `on.schedule`
If `integrations.allureServer.enabled = true`: add Allure Server publish step at end

---

## Step 4: Write YAML output

**GitHub Actions** → `<outputDir>/<projectName>/configs/.github/workflows/ui.yml`
**Azure DevOps** → `<outputDir>/<projectName>/configs/azure-pipelines.yml`
**GitLab CI** → `<outputDir>/<projectName>/configs/.gitlab-ci.yml`

---

## Step 5: Validate YAML syntax

Run: `node -e "require('js-yaml').load(require('fs').readFileSync('<path>', 'utf8'))"` — must parse without error.

---

## Generated GitHub Actions Output (Kit-U1 default, no matrix, no schedule)

```yaml
name: UI-Playwright
on:
  push:
    branches: [main]
  workflow_dispatch:

env:
  NODE_VERSION: '20'

jobs:
  smoke-test:
    name: Smoke Tests — Chromium
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install chromium --with-deps

      - name: Run smoke tests
        run: npx playwright test --grep "@smoke"
        env:
          BASE_URL: ${{ secrets.BASE_URL }}

      - name: Generate Allure report
        if: always()
        run: npx allure generate allure-results --clean -o allure-report

      - name: Upload Allure report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: allure-report
          path: allure-report
          retention-days: 30

      - name: Upload Playwright report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report
          retention-days: 30
```
