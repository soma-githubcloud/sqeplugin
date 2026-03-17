# Page Object Model Rules

Apply these rules when editing or generating `*Page.ts`, `*Page.java`, `*PO.ts`, or `*PO.java` files.

## Single Responsibility
- One Page Object per page or major UI component
- A Page Object represents ONE screen/component — not the entire application
- If a component (e.g., Navigation Bar, Modal) appears on multiple pages, create a SHARED component class

## Locator Definitions
- All locators are defined as class-level properties (TypeScript: `readonly`; Java: `private final`)
- Locators are NEVER computed at runtime from dynamic values unless absolutely required
- Naming convention (all kits): `<action><Element>` or `<elementDescription>`:
  - `emailInput`, `submitButton`, `errorMessage`, `userAvatar`, `productList`

## Action Methods — What to Include
- User-facing actions ONLY: `click`, `fill`, `select`, `hover`, `upload`, `navigate`
- Composite actions that are common and reusable (e.g., `login(email, password)`)
- Navigation method: `navigate()` using the page's URL path (not full URL — use baseURL)

## Action Methods — What to EXCLUDE
- NO assertions (`expect`, `assert`, `verify`) — these go in test specs only
- NO conditional logic based on UI state — POMs are deterministic
- NO thread sleeps or arbitrary waits — use framework wait mechanisms if needed
- NO test data generation — test data belongs in fixture files or spec setup

## Return Types
- TypeScript: all actions return `Promise<void>` or `Promise<string>` for getter methods
- Java: actions return `void`; getters return `String`, `List<String>`, `WebElement`
- Page Objects do NOT return `this` (no fluent chaining pattern — keeps things simple)

## Imports
- TypeScript: import `Page` from `@playwright/test` ONLY
- Java: import from `org.openqa.selenium.*` only; no test framework imports in POM

## File Structure Template
```typescript
// TypeScript example
export class LoginPage {
  readonly emailInput    = this.page.getByTestId('email');
  readonly passwordInput = this.page.getByTestId('password');
  readonly submitButton  = this.page.getByRole('button', { name: 'Sign in' });
  readonly errorMessage  = this.page.getByRole('alert');

  constructor(private readonly page: Page) {}

  async navigate(): Promise<void> {
    await this.page.goto('/login');
  }

  async fillCredentials(email: string, password: string): Promise<void> {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
  }

  async clickSubmit(): Promise<void> {
    await this.submitButton.click();
  }
}
```
