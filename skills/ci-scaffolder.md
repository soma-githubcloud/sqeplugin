# CI Scaffolder — Reusable Skill

Patterns and generation rules for CI/CD pipeline YAML files. Load into `agents/capability-3/ci-agent.md` for all
pipeline generation tasks. Covers GitHub Actions, Azure DevOps, and GitLab CI.

---

## Universal Pipeline Structure

All CI pipelines share this logical step sequence regardless of provider:

```
1. Checkout source code
2. Setup language runtime (Node/Java/Python)
3. Install dependencies (npm ci / mvn install / pip install)
4. Install browsers (Playwright only)
5. Run smoke tests (--grep @smoke or -m smoke)
6. Generate Allure report (always, even on failure)
7. Upload test artifacts (Allure + HTML reports)
8. (Optional) Publish to Allure Server
```

Steps 6-8 use `if: always()` or equivalent — they run even when tests fail.

---

## GitHub Actions Patterns

### Standard job structure
```yaml
jobs:
  <job-name>:
    name: <Human Name>
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: npm
      - run: npm ci
      - run: npx playwright install <browser> --with-deps
      - run: <test-command>
        env:
          BASE_URL: ${{ secrets.<ENV_VAR_NAME> }}
      - name: Generate Allure report
        if: always()
        run: npx allure generate allure-results --clean -o allure-report
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: <artifact-name>
          path: <artifact-path>
          retention-days: 30
```

### Matrix strategy (multi-browser)
```yaml
strategy:
  fail-fast: false
  matrix:
    browser: [chromium, firefox, webkit]
```
Reference in steps as: `${{ matrix.browser }}`

### Nightly schedule
```yaml
on:
  schedule:
    - cron: '0 2 * * *'   # 2 AM UTC daily
```

### Secrets reference pattern
```yaml
env:
  BASE_URL: ${{ secrets.BASE_URL }}
  API_KEY:  ${{ secrets.API_KEY }}
```
NEVER hardcode values. Always use `${{ secrets.VAR_NAME }}` for sensitive values.

---

## Azure DevOps Patterns

```yaml
trigger:
  branches:
    include: [main]

pool:
  vmImage: ubuntu-latest

steps:
  - task: NodeTool@0
    inputs:
      versionSpec: '20.x'
  - script: npm ci
    displayName: Install dependencies
  - script: npx playwright install chromium --with-deps
    displayName: Install Playwright browsers
  - script: npx playwright test --grep "@smoke"
    displayName: Run smoke tests
    env:
      BASE_URL: $(BASE_URL)
  - script: npx allure generate allure-results --clean -o allure-report
    displayName: Generate Allure report
    condition: always()
  - task: PublishPipelineArtifact@1
    displayName: Upload Allure report
    condition: always()
    inputs:
      targetPath: allure-report
      artifact: allure-report
```

---

## GitLab CI Patterns

```yaml
image: node:20

stages:
  - test
  - report

smoke-tests:
  stage: test
  script:
    - npm ci
    - npx playwright install chromium --with-deps
    - npx playwright test --grep "@smoke"
  variables:
    BASE_URL: $BASE_URL
  artifacts:
    when: always
    paths:
      - allure-results/
    expire_in: 7 days

generate-report:
  stage: report
  when: always
  script:
    - npx allure generate allure-results --clean -o allure-report
  artifacts:
    paths:
      - allure-report/
    expire_in: 30 days
```

---

## Guardrails

- **Secrets**: Always use provider's secret mechanism — NEVER hardcode tokens, URLs with credentials, or passwords in YAML
- **`if: always()`**: Report generation and artifact upload MUST run even when tests fail
- **`fail-fast: false`** on matrix jobs: one browser failing shouldn't cancel others
- **`npm ci` not `npm install`**: CI should use the lockfile exactly
- **`--with-deps`** on Playwright install: ensures OS browser dependencies are installed in CI
- **Retention**: Set artifact retention to 30 days (not forever — disk cost)
