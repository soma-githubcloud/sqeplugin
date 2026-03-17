# Gherkin Transformer — Reusable Skill

Converts structured test case objects (from `test-case-parser` skill) into valid Gherkin
`Given/When/Then` sequences. Enforces domain language and tag taxonomy. Load this into the
`agents/capability-3/bdd-agent.md` before feature file generation.

---

## Input: Structured Test Case

```json
{
  "id": "TC-001",
  "title": "Login with valid credentials",
  "tags": ["smoke"],
  "preconditions": ["User is not logged in"],
  "steps": [
    { "order": 1, "action": "navigate", "target": "login page",     "value": null },
    { "order": 2, "action": "fill",     "target": "email field",    "value": "user@test.com" },
    { "order": 3, "action": "fill",     "target": "password field", "value": "Test@123" },
    { "order": 4, "action": "click",    "target": "login button",   "value": null }
  ],
  "expectedResults": ["User is redirected to the dashboard"]
}
```

---

## Output: Gherkin Scenario

```gherkin
@ui @smoke @TC-001
Scenario: TC-001 - Login with valid credentials
  Given the user is on the login page
  When  they provide their email address
  And   they provide their password
  And   they submit the login form
  Then  they should be redirected to the dashboard
```

---

## Step Classification Rules

| Step action | Keyword | Notes |
|---|---|---|
| `navigate` | `Given` | "the user is on the <target>" |
| `precondition` text | `Given` | Direct conversion |
| `fill`, `click`, `select`, `check`, `hover`, `upload` | `When` / `And` | User actions |
| `assert`, assertion in expectedResults | `Then` / `And` | Expected outcomes |
| State assertion before actions | `Given` | "the page title is ..." |

Rules:
- First `Given` keyword is `Given`; subsequent preconditions are `And`
- First `When` keyword is `When`; subsequent actions are `And`
- First `Then` keyword is `Then`; subsequent assertions are `And`
- NEVER use `And` as the very first step in a scenario

---

## Domain Language Mapping (`strict` style)

In `strict` mode, replace UI verb phrases with business-level language:

| UI Verb Phrase | Business Language |
|---|---|
| "navigate to login page" | "the user is on the login page" |
| "fill email field with `{value}`" | "they provide their email address" |
| "fill password field with `{value}`" | "they provide their password" |
| "click login button" | "they submit the login form" |
| "click submit button" | "they submit the form" |
| "click add to cart button" | "they add the product to their cart" |
| "click checkout button" | "they proceed to checkout" |
| "fill search field with `{value}`" | "they search for `{value}`" |
| "click cancel button" | "they cancel the operation" |
| "click confirm button" / "click OK button" | "they confirm the action" |

In `relaxed` mode, use the step text as-is (without domain mapping).

---

## Tag Taxonomy (always applied)

Tags are collected from:
1. Test case `tags` array (e.g., `["smoke"]` → `@smoke`)
2. `--tags` flag from the calling command (e.g., `@regression`)
3. Always add: `@TC-<id>` (e.g., `@TC-001`)
4. Always add: `@ui` for UI kits
5. Never add `@wip` unless explicitly tagged in test case

**Tag format in feature file**:
```gherkin
@ui @smoke @TC-001 @module(login)
Scenario: TC-001 - Login with valid credentials
```

Tags are placed on the line immediately before `Scenario:`.

---

## Step Deduplication Across Scenarios

Before generating step definitions, collect ALL step texts from ALL scenarios in the batch.
Normalize: lowercase, trim whitespace, replace string literals with `{string}` placeholders.

If two steps normalize to the same pattern → merge them into ONE parameterized step:
```
"they provide their email address" (value: "user@test.com")
→ When they enter {string} as their email address
```

---

## Parameterized Scenarios

If the test case has variants (different data for same flow), generate `Scenario Outline`:

```gherkin
Scenario Outline: TC-001 - Login with <credentialType> credentials
  Given the user is on the login page
  When  they enter "<email>" as their email address
  And   they enter "<password>" as their password
  And   they submit the login form
  Then  they should be <outcome>

  Examples:
    | credentialType | email               | password   | outcome                     |
    | valid          | user@example.com    | Test@123   | redirected to the dashboard |
    | invalid        | user@example.com    | wrongpass  | shown an error message      |
```
