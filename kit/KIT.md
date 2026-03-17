# Kit-U1: UI Greenfield — Playwright TypeScript

This file is loaded by the SQE plugin when `kit/kit.config.json` has `"id": "kit-u1"`.
It provides kit-specific generation rules, naming conventions, and decision guidance.

---

## Kit Summary

| Property | Value |
|---|---|
| Tech Stack | Playwright + TypeScript |
| Test Style | Non-BDD (default) / BDD with Cucumber (optional) |
| Locator Priority | Playwright MCP → DOM Snapshot → AI-Guided |
| Reporting | Playwright HTML + Allure |
| Mode | Greenfield (creates full project structure from scratch) |

---

## TypeScript Naming Conventions

| Artifact | Convention | Example |
|---|---|---|
| Page Object class | PascalCase + `Page` suffix | `LoginPage`, `CheckoutPage` |
| Page Object file | `<ClassName>.ts` | `LoginPage.ts` |
| Spec file (non-BDD) | `<TC-ID>-<slug>.spec.ts` | `TC001-valid-login.spec.ts` |
| Feature file | `<feature-name>.feature` | `login.feature` |
| Step definition file | `<feature-name>.steps.ts` | `login.steps.ts` |
| Locator alias | camelCase | `emailInput`, `submitButton` |
| Test function name | descriptive string | `'TC-001: Login with valid credentials'` |
| Fixture file | `<name>.fixtures.ts` | `auth.fixtures.ts` |

---

## Page Object Structure (Kit-U1 Standard)

```typescript
import { Page } from '@playwright/test';

export class LoginPage {
  // ─── Locators ─────────────────────────────────────────────────
  readonly emailInput    = this.page.getByTestId('email');
  readonly passwordInput = this.page.getByTestId('password');
  readonly submitButton  = this.page.getByRole('button', { name: 'Sign in' });
  readonly errorMessage  = this.page.getByRole('alert');

  constructor(private readonly page: Page) {}

  // ─── Navigation ───────────────────────────────────────────────
  async navigate(): Promise<void> {
    await this.page.goto('/login');
  }

  // ─── Actions ──────────────────────────────────────────────────
  async fillEmail(email: string): Promise<void> {
    await this.emailInput.fill(email);
  }

  async fillPassword(password: string): Promise<void> {
    await this.passwordInput.fill(password);
  }

  async clickSubmit(): Promise<void> {
    await this.submitButton.click();
  }

  // ─── Composite Actions ────────────────────────────────────────
  async login(email: string, password: string): Promise<void> {
    await this.fillEmail(email);
    await this.fillPassword(password);
    await this.clickSubmit();
  }
}
```

---

## Non-BDD Spec Structure

```typescript
import { test, expect } from '@playwright/test';
import { LoginPage } from '@pages/LoginPage';

test.describe('Login', () => {
  test('TC-001: Login with valid credentials', async ({ page }) => {
    const loginPage = new LoginPage(page);

    await test.step('Navigate to login page', async () => {
      await loginPage.navigate();
    });

    await test.step('Enter valid credentials', async () => {
      await loginPage.login('user@example.com', 'SecurePass123');
    });

    await test.step('Verify successful login', async () => {
      await expect(page).toHaveURL(/\/dashboard/);
    });
  });
});
```

---

## BDD Structure (when testStyle = "bdd")

When `testStyle` is set to `"bdd"`, the kit uses `@cucumber/cucumber` with `@cucumber/playwright`.

**Feature file** (`features/login.feature`):
```gherkin
Feature: User Login

  @smoke @TC-001
  Scenario: TC-001 - Login with valid credentials
    Given the user is on the login page
    When they enter "user@example.com" as the email
    And they enter "SecurePass123" as the password
    And they click the sign in button
    Then they should be redirected to the dashboard
```

**Step definitions** (`steps/login.steps.ts`):
```typescript
import { Given, When, Then } from '@cucumber/cucumber';
import { expect } from '@playwright/test';
import { LoginPage } from '@pages/LoginPage';

let loginPage: LoginPage;

Given('the user is on the login page', async function () {
  loginPage = new LoginPage(this.page);
  await loginPage.navigate();
});

When('they enter {string} as the email', async function (email: string) {
  await loginPage.fillEmail(email);
});

When('they enter {string} as the password', async function (password: string) {
  await loginPage.fillPassword(password);
});

When('they click the sign in button', async function () {
  await loginPage.clickSubmit();
});

Then('they should be redirected to the dashboard', async function () {
  await expect(this.page).toHaveURL(/\/dashboard/);
});
```

---

## Playwright Config Requirements

The generated `playwright.config.ts` must:
- Set `baseURL` from `process.env.BASE_URL`
- Include both `html` and `allure-playwright` in the reporter array
- Set `testDir: './tests'` (non-BDD) or `testDir: './features'` (BDD)
- Use `fullyParallel: false` unless user explicitly requests parallel
- Include trace/screenshot/video settings for CI

---

## Strategy → Test Style Routing (Automatic)

The kit automatically selects which test artifacts to generate based on the locator strategy
resolved during intake. The `--bdd` flag always overrides this default.

| Strategy | Trigger | Default `testStyle` | Generates |
|---|---|---|---|
| `playwright-mcp` | `--url` provided | `both` | `tests/nonbdd/` specs **and** `tests/bdd/` feature + steps |
| `dom-based` | `--dom` provided | `nonbdd` | `tests/nonbdd/` specs only |
| `ai-guided` | neither provided | `nonbdd` | `tests/nonbdd/` specs only |

**Page Objects are always shared** — `src/pages/` is the single POM location regardless of
test style. Both spec files and BDD step definitions import from the same POM classes.

### Override Rules (precedence, highest first)
1. `--bdd` CLI flag → forces `testStyle = "bdd"` regardless of strategy
2. `strategyDefaults[strategy].testStyle` in `kit.config.json` → automatic routing
3. Top-level `testStyle` in `kit.config.json` → fallback if strategy not matched

### Output tree — `playwright-mcp` (both styles)
```
outputs/<project>/
  src/pages/          ← shared POMs (always here)
  tests/
    nonbdd/           ← spec files (.spec.ts)
    bdd/              ← feature files (.feature) + step definitions (.steps.ts)
```

### Output tree — `dom-based` or `ai-guided` (nonbdd only)
```
outputs/<project>/
  src/pages/          ← shared POMs
  tests/
    nonbdd/           ← spec files only
    bdd/              ← directory exists but is NOT populated
```
