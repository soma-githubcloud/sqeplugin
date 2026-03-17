# BDD Gherkin Rules

Apply these rules when editing or generating `*.feature` files.

## Structure
- Every feature file has exactly ONE `Feature:` declaration
- Feature title matches the page or user journey being tested
- Each `Scenario:` title must be unique within the file
- Scenario titles must be traceable to test case IDs (e.g., `TC-001: Login with valid credentials`)

## Step Keywords
- Use ONLY: `Given`, `When`, `Then`, `And`, `But`
- NEVER start a scenario with `And` or `But` as the first step — always begin with `Given`
- `Given` = precondition/state setup
- `When` = user action
- `Then` = assertion/expected outcome
- `And` / `But` = continuation of the previous keyword's type

## Step Granularity
- Each `Then` step contains ONE assertion — do NOT combine multiple assertions in one step
- Steps must be reusable across scenarios — avoid embedding specific data directly in step text
- Use `<angle brackets>` or tables for parameterized steps:
  ```gherkin
  When I enter "<email>" in the email field
  And I enter "<password>" in the password field
  ```

## Scenario Outline
- Use `Scenario Outline:` + `Examples:` table when the same flow needs multiple data sets
- Column headers in Examples must match `<placeholder>` names in steps exactly

---

## Tag Taxonomy (Enforced)

All feature files and scenarios MUST use tags from this taxonomy. No free-form tags outside
this list are permitted.

| Tag | Scope | Meaning |
|---|---|---|
| `@ui` | Feature | All UI test scenarios |
| `@api` | Feature | All API test scenarios |
| `@smoke` | Scenario | Fast, critical-path tests run on every CI push |
| `@regression` | Scenario | Full regression suite (not required on every push) |
| `@module(<name>)` | Scenario | Module/domain grouping, e.g. `@module(login)`, `@module(checkout)` |
| `@TC-<id>` | Scenario | Traceability to test case ID, e.g. `@TC-001`, `@TC-042` |
| `@wip` | Scenario | Work in progress — EXCLUDED from CI runs by default |
| `@negative` | Scenario | Negative / error-path test |
| `@boundary` | Scenario | Boundary value test |

### Tag Placement Rules
- `@ui` or `@api` → on the `Feature:` line
- All others → on the `Scenario:` or `Scenario Outline:` line immediately above the keyword
- `@TC-<id>` is MANDATORY for every scenario — enables test case traceability
- `@smoke` and `@regression` are mutually exclusive on the same scenario

### Example (correct tags)
```gherkin
@ui
Feature: User Login

  @smoke @module(login) @TC-001
  Scenario: TC-001 - Login with valid credentials
    Given the user is on the login page
    When they submit valid credentials
    Then they should be redirected to the dashboard

  @regression @negative @module(login) @TC-002
  Scenario: TC-002 - Login with invalid password
    Given the user is on the login page
    When they submit an incorrect password
    Then an error message should be displayed

  @wip @module(login) @TC-003
  Scenario: TC-003 - Login with SSO
    Given the user selects SSO login
    When they authenticate with their organization
    Then they should be granted access
```

---

## Domain Language Guard (Strict Mode)

In **strict mode** (default for production feature files), step wording must use business/domain
language. UI implementation details are forbidden in step text.

### Forbidden in steps (strict mode)
| Forbidden pattern | Use instead |
|---|---|
| `click the login button` | `submit the login form` |
| `fill in the email field` | `provide their email address` |
| `click the checkbox` | `agree to the terms` |
| `press Enter` | `confirm the action` |
| `navigate to /login` | `the user is on the login page` |
| Element IDs, CSS selectors, XPath | Never appear in step wording |

### Relaxed Mode
Use `--style relaxed` with generation when AI-guided tier is active and precise UI verbs improve
step clarity. Even in relaxed mode, never embed selectors or element IDs in steps.

---

## Selector Lint (in *.feature step text)
These patterns trigger a lint warning:
- `id=ember[0-9]+` — dynamic Ember.js IDs
- `class=css-[a-z0-9-]+` — CSS-in-JS generated class names
- Any XPath expression in step wording
- Hardcoded URLs (e.g., `https://`, `http://`)

---

## Anti-patterns to avoid
- Do NOT: `Then the login is successful and the dashboard is shown` (two assertions in one Then)
- Do NOT: `When I navigate to http://app.com/login` (hardcode URLs — use step definitions)
- Do NOT: `And click login` without a preceding `When` step
- Do NOT: use `@critical` — use `@smoke` or `@regression` instead
- Do NOT: omit `@TC-<id>` — every scenario must trace to a test case
