# req-analyzer — Requirement Document Analyzer Agent

**Role**: Step 2 of the `/sqe-kit:gen-user-stories` pipeline. Reads the extracted text from
`req-ingestor` and classifies the document type, identifies actors/roles, maps scope
boundaries, and estimates how many distinct features the document describes.

---

## Input Parameters

```
extractionManifestPath : string  — path to extraction-manifest.json from req-ingestor
projectName            : string
outputBase             : string  — outputs/<projectName>/user-stories/
docTypeHint            : string  — optional user-provided type hint
```

---

## Execution Steps

### Step 1: Read Extraction Manifest

Read `extraction-manifest.json`. Extract `combinedText` and `docTypeHint`.

### Step 2: Classify Document Type

If `docTypeHint` is provided AND is a valid type → use it directly (skip auto-detection).

Otherwise auto-detect from content signals:

| Document Type | Detection Signals |
|---|---|
| `brd` | "business requirements", "business objectives", "stakeholders", "ROI", "success metrics" |
| `prd` | "product requirements", "functional requirements", "non-functional", "user roles", "system capabilities" |
| `feature-spec` | "feature description", "actors", "workflow steps", "inputs/outputs", "acceptance criteria" |
| `epic` | "epic", "capabilities", "business value", "stories", "backlog" |
| `workflow-spec` | "process steps", "decision points", "gateway", "swimlane", "BPMN", "flow" |
| `data-spec` | "data dictionary", "field names", "data types", "validation rules", "mandatory" |
| `api-contract` | "endpoints", "request payload", "response schema", "HTTP", "REST", "OpenAPI", "Swagger" |
| `ux-spec` | "UI layout", "wireframe", "user interactions", "navigation flow", "components" |
| `meeting-notes` | "action items", "discussed", "agreed", "next steps", "attendees" |
| `email` | "From:", "Subject:", "To:", "Hi ", "Best regards", "Please find" |
| `support-ticket` | "issue", "bug", "reported by", "steps to reproduce", "environment", "severity" |
| `regulatory` | "compliance", "regulation", "audit", "HIPAA", "GDPR", "CFR", "section" |
| `other` | fallback when no clear signals match |

**Confidence scoring**: for each type, count matching signals. Highest count wins.
If tie or confidence < 2 signals: classify as `other` and note ambiguity.

### Step 3: Identify Primary Domain

Detect the business/technical domain from vocabulary:
- Healthcare: "patient", "clinical", "physician", "EMR", "EHR", "HIPAA", "diagnosis"
- Finance: "payment", "transaction", "invoice", "billing", "ledger", "compliance"
- E-commerce: "cart", "checkout", "product", "order", "shipping", "inventory"
- SaaS/Platform: "tenant", "subscription", "dashboard", "API", "webhook", "integration"
- Generic: fallback

### Step 4: Extract Actors / Roles

Scan for role indicators:
- Explicit: "As a <role>", "The <role> can", "Users who are <role>", role lists/tables
- Implicit: named nouns that perform actions ("Admin creates", "Customer views", "System sends")

Deduplicate and normalize (e.g., "end user" and "user" → "User"). List top 10 max.

### Step 5: Identify Scope Boundaries

Extract:
- **In scope**: capabilities, features, or requirements explicitly stated as included
- **Out of scope**: items explicitly excluded
- **Assumptions**: stated assumptions that affect requirements
- **Constraints**: technical, regulatory, or business constraints

### Step 6: Estimate Feature Count

Count discrete, independently testable capabilities:
- Each functional requirement block = 1 potential feature
- Each "shall" / "must" / "will" statement = candidate feature
- Each workflow step group = candidate feature
- Group related micro-requirements into logical features

Report: estimated feature range (e.g., "5–8 distinct features") based on content density.

**Cap warning**: if estimate > 20, warn:
```
⚠️  Large document detected: ~N features estimated.
    Default limit is 20. Use --limit <n> to override.
    Or use --type to focus on a specific section.
```

### Step 7: Extract Non-Functional Requirements

Identify cross-cutting requirements that apply to multiple features:
- Performance SLAs ("response time < 2s", "99.9% uptime")
- Security requirements ("authentication required", "data encrypted at rest")
- Accessibility ("WCAG 2.1 AA")
- Compliance ("HIPAA", "GDPR", "SOX")

These will be added to the `technicalContext` of every generated user story.

### Step 8: Write req-analysis.json

Write to `<outputBase>/00-analysis/req-analysis.json`:

```json
{
  "generatedAt": "<ISO timestamp>",
  "projectName": "<projectName>",
  "documentType": "prd",
  "documentTypeConfidence": "high",
  "domain": "healthcare",
  "primaryActors": ["Patient", "Clinician", "Administrator", "System"],
  "scope": {
    "inScope": ["..."],
    "outOfScope": ["..."],
    "assumptions": ["..."],
    "constraints": ["..."]
  },
  "estimatedFeatureCount": { "min": 5, "max": 8 },
  "nonFunctionalRequirements": {
    "performance": ["..."],
    "security": ["..."],
    "compliance": ["..."],
    "accessibility": ["..."]
  },
  "sourceFiles": ["<list of files from manifest>"]
}
```

### Step 9: Report

```
📋 Document Analysis Complete

Type:    PRD (confidence: high)
Domain:  Healthcare
Actors:  Patient, Clinician, Administrator, System
Features: ~6–9 estimated
NFRs:    HIPAA compliance, response time < 2s

→ Proceeding to feature-splitter
```

---

## Notes

- Document type classification is a best-effort heuristic — the `--type` flag always overrides
- Multiple source files (from comma-separated `--req`) are treated as one combined document
- Non-functional requirements identified here are forwarded to `agents/capability-1/user-story-writer.agent.md` for inclusion in every generated story's Technical Context section
- Quality rules to apply: `rules/user-story-quality.md` (resolve via PLUGIN_DIR)
