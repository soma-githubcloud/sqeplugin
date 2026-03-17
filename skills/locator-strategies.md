# Locator Strategies — Reusable Knowledge

This skill provides the canonical rules for selecting, ranking, and generating locators across
all kits. Load this into any agent or command that deals with UI element identification.

---

## Universal Locator Priority

Apply this priority order for ALL kits. Higher = more preferred.

| Priority | Strategy | Stability | Example |
|---|---|---|---|
| 1 | `data-testid` attribute | Highest — explicit test marker | `[data-testid="submit-btn"]` |
| 2 | `data-cy`, `data-test`, `data-qa` | High — purpose-built test attrs | `[data-cy="email-input"]` |
| 3 | ARIA role + accessible name | High — semantic, stable | `role=button[name="Sign in"]` |
| 4 | ARIA label | Medium-high | `[aria-label="Close dialog"]` |
| 5 | Form element `name` attr | Medium | `[name="username"]` |
| 6 | Placeholder text | Medium | `[placeholder="Enter email"]` |
| 7 | Unique CSS `id` | Medium — often auto-generated | `#loginBtn` |
| 8 | CSS class (stable, BEM) | Low — can change with styling | `.login-form__submit` |
| 9 | Element text content | Low — fragile to copy changes | `text="Submit"` |
| 10 | XPath | Lowest — last resort only | `//button[@type='submit']` |

**NEVER USE**: positional selectors (`:nth-child`, `:eq()`), parent-dependent selectors, absolute XPath.

---

## Playwright Locator API Mapping

```
Priority 1-2:  page.getByTestId('value')
Priority 3:    page.getByRole('button', { name: 'Sign in' })
               page.getByRole('textbox', { name: 'Email' })
               page.getByRole('combobox', { name: 'Country' })
Priority 4:    page.getByLabel('Email address')
Priority 5-6:  page.getByPlaceholder('Enter your email')
Priority 7-8:  page.locator('#loginBtn')
               page.locator('.submit-button')
Priority 9:    page.getByText('Submit', { exact: true })
Priority 10:   page.locator('xpath=//button[@type="submit"]')
```

**Playwright best practice**: When `getByTestId` isn't available, prefer `getByRole` as it tests
accessibility simultaneously.

---

## Locator Naming Conventions

| Element Type | TypeScript Name |
|---|---|
| Text input | `emailInput`, `passwordInput` |
| Submit button | `submitButton`, `loginButton` |
| Link | `forgotPasswordLink` |
| Dropdown | `countrySelect`, `roleDropdown` |
| Checkbox | `rememberMeCheckbox` |
| Radio button | `adminRoleRadio` |
| Error/alert | `errorMessage`, `validationAlert` |
| Success message | `successBanner`, `confirmationText` |
| Table | `usersTable`, `productGrid` |
| Modal | `confirmModal`, `deleteDialog` |
| Page heading | `pageTitle`, `sectionHeading` |

---

## data-testid Normalization

When a `data-testid` value is found, normalize to `camelCase` for the locator alias:
- `data-testid="email-input"` → alias: `emailInput`
- `data-testid="submit-btn"` → alias: `submitButton`
- `data-testid="user-avatar"` → alias: `userAvatar`
- `data-testid="error-message"` → alias: `errorMessage`

---

## Common Traps to Avoid

1. **Dynamic IDs**: `id="ember123"`, `id="react-select-2"` — NEVER use these; they change each render
2. **Auto-generated classes**: `class="css-1a2b3c"` — Tailwind/CSS-in-JS classes; avoid
3. **Text with internationalization**: `text="Submit"` breaks in non-English locales
4. **Parent-child chains**: `.container > .row > .col > button` — too brittle
5. **Index-based**: `.items:nth-child(3)` — breaks when DOM order changes
