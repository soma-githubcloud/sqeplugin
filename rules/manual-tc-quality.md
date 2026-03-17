# Manual Test Case Quality Rules

Apply these rules when generating or reviewing manual test cases (both BDD and non-BDD formats)
produced by the story-to-testcase pipeline.

---

## Universal Rules (Both BDD and Non-BDD)

### Traceability
- Every test case MUST have a unique TC ID (`TC-NNN` format, zero-padded to 3 digits)
- Every TC must trace to at least one user story acceptance criterion
- If no explicit AC exists, cite the user story goal or technical requirement
- GAP-filled scenarios must include `GAP-{id}` reference comment

### Specificity
- Test data must be concrete and realistic — no placeholders like `validEmail@test.com` or `testUser`
- Expected results must be verifiable — "error message displayed" is NOT acceptable;
  "error message 'Invalid email format' is displayed" IS acceptable
- Numeric values must be exact: "8 characters" not "short password", "500ms" not "fast"

### Coverage
- Every acceptance criterion from the user story must be covered by at least one TC
- At least one positive, one negative, and one edge case TC must exist per functional area
- Security TCs are mandatory when user story includes authentication, authorization, or user data

### Completeness — Non-BDD Format
- `TC ID` field: required
- `Title` field: required — must describe what is being tested in plain English
- `Preconditions`: required (can be "None" if truly no preconditions)
- `Test Steps`: required — numbered, each step is one discrete action
- `Expected Results`: required — must match step count (one result per step or summary)
- `Test Data`: required if steps reference specific values; "N/A" if no specific data

### Completeness — BDD Format
- `@TC-NNN` tag: mandatory on every scenario
- `@ui` or `@api` tag: mandatory on every `Feature:` declaration
- `@smoke` or `@regression` tag: mandatory on every scenario (mutually exclusive)
- `@module(<name>)` tag: mandatory on every scenario
- `@negative` tag: mandatory on all negative and security attack scenarios
- `@boundary` tag: mandatory on all edge case scenarios

---

## Anti-Patterns to Avoid

### Non-BDD Anti-Patterns
- Steps that contain multiple actions: "Enter email and password and click login" → split into 3 steps
- Expected results that say "Test passes" — must state what specifically passes
- Preconditions that are actually test steps (actions, not states)
- Vague test data: "a valid password" → use "SecurePass123!" with explanation

### BDD Anti-Patterns (per `bdd-gherkin.md`)
- Starting a scenario with `And` or `But`
- Multiple assertions in one `Then` step → split into `Then` + `And`
- Raw selectors in step wording (`#loginBtn`, `.css-abc123`, `id=ember42`)
- Hardcoded URLs in step wording
- `@critical` tag → use `@smoke` or `@regression` instead
- Omitting `@TC-<id>` → every scenario must have it
- Using same tag for both `@smoke` and `@regression`

---

## Naming Conventions

### TC Titles (Non-BDD)
Pattern: `[Subject] [Action Outcome] [Condition]`
- Good: "User successfully registers with valid credentials"
- Good: "Registration fails when email format is invalid"
- Bad: "Test email validation"
- Bad: "Check login"

### Scenario Titles (BDD)
Include the TC ID:
- `TC-001 — User successfully registers with valid credentials`
- `TC-020 — Registration fails when email format is invalid`

### Module Names (BDD `@module`)
Use lower-camelCase or kebab-case nouns from the feature domain:
- `@module(registration)`, `@module(login)`, `@module(checkout)`, `@module(payment)`
- Not: `@module(test1)`, `@module(myFeature)`, `@module(TC001)`

---

## Quality Score Thresholds

After TC formatter runs, the following minimums must be met before handoff to automation:

| Metric | Minimum |
|--------|---------|
| Acceptance criteria coverage | 80% |
| TCs with valid test data | 100% |
| TCs with specific expected results | 100% |
| BDD scenarios with `@TC-NNN` tag | 100% |
| BDD scenarios with `@smoke` or `@regression` | 100% |

If any minimum is not met, flag the TC document with a lint warning before handing off.

---

## Selector Lint (BDD Only)

These patterns in BDD step text trigger a lint warning:
- `id=ember[0-9]+` — dynamic Ember.js IDs
- `class=css-[a-z0-9-]+` — CSS-in-JS generated classes
- Any XPath expression in step wording
- Hardcoded `https://` or `http://` URLs in step wording
