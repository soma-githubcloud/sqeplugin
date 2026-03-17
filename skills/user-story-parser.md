# Skill: User Story Parser

Apply this skill to extract structured information from user stories in any common format before
passing them to specialist agents.

## Supported Input Formats

### Format 1: Agile Standard
```
As a <role>
I want <goal>
So that <benefit>

Acceptance Criteria:
1. ...
2. ...
```

### Format 2: Plain English Feature Description
```
# Feature: User Registration
[Description paragraphs]
[Acceptance criteria as bullet points or numbered list]
```

### Format 3: Structured Specification
```
## Feature Name
**User Story**: ...
**Description**: ...
**Acceptance Criteria**:
  - AC1: ...
  - AC2: ...
**Technical Notes**: ...
```

### Format 4: Inline / Mixed
Free-form text that includes user roles, goals, and criteria in any order.

## Parsing Steps

### Step 1: Detect Format
Look for:
- "As a" → Format 1
- "# Feature:" or "## Feature" → Format 2 or 3
- Numbered/bulleted list under "Acceptance" → any format with AC

### Step 2: Extract Core Fields

| Field | Where to find it |
|-------|-----------------|
| `featureName` | Document title, "Feature:" heading, or first heading |
| `role` | "As a <role>" or "The <role>" in description |
| `goal` | "I want <X>" or the primary action described |
| `benefit` | "So that <Y>" or the business value stated |
| `acceptanceCriteria` | Numbered/bulleted list under "Acceptance Criteria:", "AC:", "Given that", or similar |
| `technicalContext` | API specs, DB mentions, service names, performance SLAs, security requirements |
| `constraints` | Non-functional requirements, business rules, regulatory requirements |
| `explicitEdgeCases` | Any section titled "Edge Cases", "Corner Cases", "Exceptions" |

### Step 3: Build Structured Object

```json
{
  "featureName": "string",
  "slug": "kebab-case-feature-name",
  "role": "string",
  "goal": "string",
  "benefit": "string",
  "acceptanceCriteria": [
    { "id": "AC1", "text": "..." },
    { "id": "AC2", "text": "..." }
  ],
  "technicalContext": {
    "hasApi": true,
    "apiEndpoints": ["POST /api/v1/auth/register"],
    "hasDatabase": true,
    "externalServices": ["SendGrid", "PostgreSQL"],
    "hasUi": true,
    "uiComponents": ["registration form", "password strength indicator"],
    "performanceSLAs": ["API response < 500ms", "100 concurrent users"],
    "securityRequirements": ["bcrypt hashing", "rate limiting", "JWT"]
  },
  "constraints": ["string"],
  "explicitEdgeCases": ["string"],
  "rawText": "full original user story text"
}
```

### Step 4: Validate Completeness

Flag warnings (do not stop processing):
- `featureName` missing → use "Unknown Feature"
- `acceptanceCriteria` empty → warn "No acceptance criteria found — specialists will rely on description only"
- `goal` missing → warn "No user goal identified — inferring from description"

## Usage

This skill is invoked by `quality-master-orchestrator` before dispatching to specialist agents.
The parsed object is passed as context alongside `rawText`.

When passing to specialists: always pass `rawText` (complete user story) — not the parsed
object. The parsed object is used internally by orchestrators for routing and reporting.

## Slug Generation

```
featureName → lowercase → replace spaces and special chars with hyphens → trim hyphens
"User Registration & Login" → "user-registration-login"
"Shopping Cart (v2)" → "shopping-cart-v2"
```
