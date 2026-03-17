# Skill: TC Formatter — Non-BDD

Apply this skill to transform raw Gherkin scenarios (from specialist agents and gap-filler) into
structured **non-BDD test case documents** — the traditional TC-ID / Steps / Expected Result
format used in test management tools (JIRA, TestRail, Azure Test Plans, etc.).

These output files serve dual purpose:
1. **Manual test case documentation** — imported into test management tools or reviewed by QA
2. **Automation input** — fed to `intake-agent` → `testgen-agent` when `handoffToAutomation = true`

## Input

```
scenarioSources : array of file paths (Phase 1 + Phase 3 output)
projectName     : string
featureSlug     : string   — kebab-case feature name
```

## Output

One or more `.md` files in the appropriate `manual-tcs/nonbdd/` directory.

## Gherkin → TC Conversion Rules

### Mapping

| Gherkin element | TC document field |
|-----------------|------------------|
| Feature name | Test Suite / Feature Name |
| Scenario name | TC Title |
| Given steps | Preconditions |
| When + And steps | Test Steps (numbered list) |
| Then + And steps | Expected Results (numbered list, matching step count) |
| Feature description | Test Suite Description |

### TC ID Assignment

Assign sequential IDs: TC-001, TC-002, TC-003 ...
Group IDs by test type:
- TC-001 to TC-019: Positive
- TC-020 to TC-039: Negative
- TC-040 to TC-059: Edge Cases
- TC-060 to TC-079: Integration
- TC-080 to TC-099: Security
- TC-100 to TC-119: Performance
- TC-120 to TC-139: API
- TC-140 to TC-159: UI/UX

### Priority Assignment

| Test Type | Default Priority |
|-----------|-----------------|
| Positive | High |
| Negative | High |
| Security | Critical |
| API | High |
| Edge Cases | Medium |
| Integration | Medium |
| Performance | Medium |
| UI/UX | High |

## Output File Format

```markdown
# Test Cases: [Feature Name]

**Project**: [projectName]
**Feature**: [featureName]
**Generated**: [timestamp]
**Total TCs**: [N]

---

## Positive Test Cases

---

### TC-001: [Scenario Title]

| Field | Value |
|-------|-------|
| **TC ID** | TC-001 |
| **Title** | [Scenario title — what is being tested] |
| **Type** | Positive |
| **Priority** | High |
| **Module** | [module name inferred from feature] |

**Description**: [Feature description or scenario context — 1 sentence]

**Preconditions**:
1. [First Given step converted to plain English]

**Test Steps**:

| Step | Action |
|------|--------|
| 1 | [First When step in plain English] |
| 2 | [Second When step in plain English] |

**Expected Results**:

| Step | Expected Result |
|------|----------------|
| 1 | [Corresponding Then/And step] |
| 2 | [Next expected result] |

**Test Data**:
- Email: john.doe@example.com
- Password: SecurePass123!

---
```

## Plain English Conversion

When converting Gherkin steps to plain English:
- Remove Given/When/Then/And keywords
- Capitalize first word
- Rephrase passive Gherkin steps into active instructions:
  - "the user enters 'john@example.com' in the email field" → "Enter 'john@example.com' in the email field"
  - "the API returns HTTP 201 Created" → "Verify the API returns HTTP 201 Created"
  - "an error message is displayed" → "Verify that an error message is displayed"

For Preconditions (Given → state, not action):
- Keep as state descriptions, not instructions:
  - "the user is on the registration page" → "User is on the registration page"
