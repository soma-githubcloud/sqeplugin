# Playwright TypeScript Rules

Apply these rules when editing or generating `*.spec.ts`, `*.test.ts`, or `*.page.ts` files.

## Imports
- Use ONLY `@playwright/test` for test primitives: `import { test, expect, Page } from '@playwright/test'`
- Page Objects are imported with relative paths: `import { LoginPage } from '../pages/LoginPage'`
- Never import from `playwright` directly (use `@playwright/test` wrapper)

## Page Object Classes
- Page classes are plain TypeScript classes — do NOT extend any base class
- Constructor signature: `constructor(private readonly page: Page) {}`
- All locators are `readonly` class properties defined at class level:
  ```typescript
  readonly emailInput = this.page.getByTestId('email');
  readonly submitButton = this.page.getByRole('button', { name: 'Submit' });
  ```
- Locator priority (highest to lowest): `getByTestId` > `getByRole` > `getByLabel` > `getByPlaceholder` > CSS with `data-*` > CSS with `id` > CSS with `class`
- NEVER use: XPath selectors, `page.$`, `page.$$`, positional selectors (`:nth-child`)

## Action Methods
- All action methods return `Promise<void>` — they do NOT return page objects or strings
- Naming: `click<Element>()`, `fill<Field>(value: string)`, `select<Field>(option: string)`
- Wrap complex interactions in `await test.step('description', async () => { ... })`
- No assertions inside Page Objects — they belong in spec files only

## Spec Files
- Each spec file tests ONE user scenario or one page's flows
- File naming: `<feature>.spec.ts` (e.g., `login.spec.ts`, `checkout.spec.ts`)
- Every `test()` must call `test.step()` for each logical action:
  ```typescript
  test('TC-001: Login with valid credentials', async ({ page }) => {
    const loginPage = new LoginPage(page);
    await test.step('Navigate to login', () => loginPage.navigate());
    await test.step('Enter credentials', () => loginPage.fillCredentials('user@test.com', 'pass'));
    await test.step('Submit form', () => loginPage.clickSubmit());
    await expect(page).toHaveURL(/dashboard/);
  });
  ```

## Waits and Timing
- NEVER use `page.waitForTimeout()` or `setTimeout()`
- Use `await expect(locator).toBeVisible()` to wait for elements
- For navigation: `await page.waitForURL(/pattern/)`
- For network: use `page.waitForResponse()` if needed

## Configuration
- `playwright.config.ts` must define `baseURL`, `testDir`, `reporter` array
- Always include both `'html'` and `['allure-playwright']` in reporter array
- Use `fullyParallel: false` for tests with shared state

## Assertions
- Use `expect` from `@playwright/test` only
- Prefer: `toBeVisible()`, `toHaveText()`, `toHaveValue()`, `toHaveURL()`, `toBeEnabled()`
- Never assert on exact pixel values or dynamic IDs
