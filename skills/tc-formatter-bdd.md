# Skill: TC Formatter — BDD

Apply this skill to transform raw Gherkin scenarios (from specialist agents and gap-filler) into
properly tagged, kit-compliant `.feature` files for use as **manual test cases**.

These output files serve dual purpose:
1. **Manual test case documentation** — reviewed and executed by QA engineers
2. **Automation input** — fed to `bdd-agent` when `handoffToAutomation = true`

## Input

```
scenarioSources : array of file paths (Phase 1 + Phase 3 output)
projectName     : string
featureSlug     : string   — kebab-case feature name (from user-story-parser)
testStyle       : string   — must include "bdd" to trigger this skill
```

## Output

One or more `.feature` files in the appropriate `manual-tcs/bdd/` directory.

## Transformation Rules

### Step 1: Merge and Deduplicate Scenarios

1. Read all input `.feature` files (Phase 1 + Phase 3 gap-filled)
2. Group scenarios by test type: positive, negative, edge-cases, integration, security,
   performance, api, ui-ux
3. Deduplicate: if a gap-filled scenario is functionally identical to an existing Phase 1
   scenario, keep the gap-filled version (it has better reasoning)

### Step 2: Apply Kit Tagging Rules

Per `rules/bdd-gherkin.md` rules:

**Feature-level tag** (place on the `Feature:` line):
- If scenarios cover UI → add `@ui`
- If scenarios cover API only → add `@api`
- If mixed → add `@ui` (UI takes precedence)

**Scenario-level tags** (place above each `Scenario:` or `Scenario Outline:`):

| Tag | When to apply |
|-----|--------------|
| `@TC-<NNN>` | Every scenario — MANDATORY. Auto-generate sequential IDs: TC-001, TC-002, ... |
| `@smoke` | Positive scenarios + primary API contract scenarios |
| `@regression` | All other scenarios (negative, edge, security, performance, integration) |
| `@module(<name>)` | Group by functional area: `@module(registration)`, `@module(login)`, etc. |
| `@negative` | All negative-scenarios and security attack scenarios |
| `@boundary` | All edge-cases scenarios |
| `@wip` | Do NOT add unless explicitly requested |

**Mutual exclusion**: `@smoke` and `@regression` cannot both appear on the same scenario.
If a scenario qualifies for both, use `@smoke`.

### Step 3: Apply Domain Language (Strict Mode)

Per `rules/bdd-gherkin.md` strict mode rules, replace UI implementation language with domain language:

| Replace | With |
|---------|------|
| "click the login button" | "submit the login form" |
| "fill in the email field" | "provide their email address" |
| "navigate to /login" | "the user is on the login page" |
| "press Enter" | "confirm the action" |
| "click the checkbox" | "agree to the terms" |

Only apply where the replacement is semantically equivalent. In API scenarios, keep technical
step wording (HTTP methods, status codes, endpoint paths are acceptable).

### Step 4: Normalize Structure

Each scenario must:
- Start with `Given` (never `When`, `And`, or `But` as first step)
- Have at least one `When` step
- Have at least one `Then` step
- Use `And` / `But` for continuations (not repeated `Given`/`When`/`Then`)
- One assertion per `Then` step — split multi-assertion `Then` steps

### Step 5: Assign TC IDs

Generate sequential TC IDs starting from TC-001 across all scenarios in the output files.
Format: `TC-<3-digit-zero-padded-number>` (TC-001, TC-002, ... TC-099, TC-100, TC-101).

Include the TC ID in the scenario title:
```gherkin
@TC-001 @smoke @module(registration)
Scenario: TC-001 — Successful registration with valid credentials
```

### Step 6: Output File Structure

```gherkin
@ui
Feature: [Feature Name]
  [Business description — 1-2 sentences explaining what this feature does]

  # ==========================================
  # POSITIVE SCENARIOS
  # ==========================================

  @TC-001 @smoke @module(registration)
  Scenario: TC-001 — [Descriptive title — what succeeds]
    Given [precondition in domain language]
    When [user action in domain language]
    Then [expected outcome]

  # ... continue for negative, edge cases, integration, security, performance, API, UI ...
```

### Step 7: Save Output Files

Naming convention: `<featureSlug>.feature` for a single feature, or
`<featureSlug>-<testType>.feature` if splitting large feature files by type (> 20 scenarios).

**Also save a TC index**: `TC-INDEX.md`

```markdown
# Test Case Index — BDD

| TC ID | Title | Type | Tags | File |
|-------|-------|------|------|------|
| TC-001 | Successful registration | Positive | @smoke @module(registration) | registration.feature:L18 |
```

## Validation Checklist

Before saving, verify:
- [ ] Every scenario has `@TC-<NNN>` tag
- [ ] No scenario has both `@smoke` and `@regression`
- [ ] No scenario starts with `And` or `But`
- [ ] `@ui` or `@api` is present on every `Feature:` declaration
- [ ] Each `Then` step contains exactly one assertion
- [ ] No raw CSS selectors, XPath, or element IDs in step wording
- [ ] No hardcoded URLs in step wording
