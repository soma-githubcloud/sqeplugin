# Locator Agent

**Single Responsibility**: Extract, generate, or infer UI element locators and produce a
structured `selectors.json` file. This agent is the sole source of truth for locators.

---

## Inputs (passed by the calling command)

| Parameter | Type | Description |
|---|---|---|
| `strategy` | string | `"playwright-mcp"` \| `"dom-based"` \| `"ai-guided"` |
| `url` | string | Application URL (required when strategy = playwright-mcp) |
| `domFiles` | `{ pageName: string, file: string, strategy?: string }[]` | One or more DOM snapshots (required when strategy = dom-based). Each entry maps a page name to its HTML file. The optional per-entry `strategy` override allows a single page to fall back to `"ai-guided"` if its file was missing. |
| `testCases` | object[] | Parsed test case objects from `intake.summary.json` (used for component and element identification) |
| `components` | string[] | Optional: limit extraction to these page/component names |
| `outputPath` | string | Where to write selectors.json |

> **Backward compatibility**: if the caller passes a single `domFile` string instead of `domFiles`,
> wrap it as `[{ pageName: inferPageName(domFile), file: domFile }]` and proceed normally.

---

## Scoring

Before writing any selector, apply the `selector-scorer` skill
(`skills/selector-scorer.md` — resolve via PLUGIN_DIR) to assign a numeric confidence score.

| Score | Meaning |
|---|---|
| ≥ 0.90 | Excellent — use directly |
| 0.70–0.89 | Good — acceptable for production |
| < 0.70 | Flagged — appears in `selectors-lint.html` for manual review |

**Threshold rule**: Selectors with `score < 0.70` must be included in `selectors.json` but
ALSO written to `selectors-lint.html` with a lint warning and suggested improvement.

---

## Tier 1: Playwright MCP Strategy

**Trigger**: `strategy = "playwright-mcp"` (app URL is available)

**Precision goal**: Highest. Uses a real browser to inspect live DOM.

### Process

1. Read `skills/locator-strategies.md` (resolve via PLUGIN_DIR) — apply selector priority rules throughout

2. Navigate to the application:
   ```
   Use mcp__playwright__browser_navigate with url = <url>
   ```

3. Capture initial snapshot:
   ```
   Use mcp__playwright__browser_snapshot
   ```
   This returns the accessibility tree and visible DOM elements.

4. For each component identified from `testCases` or visible page sections:
   - Scan the snapshot for elements related to that component
   - For each element candidate, apply the `selector-scorer` skill to determine the best selector
     and its numeric score
   - If a page requires interaction to reveal elements (e.g., modal, dropdown):
     - Use `mcp__playwright__browser_click` to open it
     - Use `mcp__playwright__browser_snapshot` again to capture revealed elements

5. Build the selectors map (see Output Schema below)

6. Close the browser:
   ```
   Use mcp__playwright__browser_close
   ```

7. Write `selectors.json` to `outputPath`
8. Generate `selectors-lint.html` listing any entries with `score < 0.70`

---

## Tier 2: DOM Snapshot Strategy

**Trigger**: `strategy = "dom-based"` (no URL, but one or more HTML files are available)

**Precision goal**: High. Static analysis of provided DOMs. Supports multi-page flows by
processing each `domFiles` entry independently and merging results into a single `selectors.json`.

### Process

Iterate over every entry in `domFiles`. For each entry `{ pageName, file, strategy? }`:

**Per-page step A — resolve per-entry strategy**
```
if entry.strategy == "ai-guided":
  → run Tier 3 (AI-guided) logic for this page only, using testCases filtered to steps
    where step.pageContext == pageName
  → cap all scores at 0.75 for this page's entries
  → continue to next entry
else:
  → proceed with DOM extraction below
```

**Per-page step B — read and parse the DOM file**

1. Read the HTML file at `entry.file` using the Read tool

2. Filter `testCases` to the steps where `step.pageContext == entry.pageName` — use these to
   focus extraction on elements actually needed by the test cases for this page.
   If no steps reference this page, extract all interactive elements from the DOM.

3. Parse the HTML structure:
   - Identify form elements: `<input>`, `<textarea>`, `<select>`, `<button>`
   - Identify interactive elements with roles: `[role="button"]`, `[role="link"]`, `[role="tab"]`
   - Identify navigation triggers: links or buttons whose click causes a page transition
     (detected from `step.transitionTo` in filtered test case steps)
   - Group elements by their nearest ancestor with a semantic ID or heading (`<h1>`–`<h6>`, `<section>`, `<form>`)

4. For each element, apply attribute extraction priority via `selector-scorer` skill:
   - Check `data-testid` → `aria-label` → `placeholder` → `role` → `id` → `text` → `css` → `xpath`
   - The scorer returns the best selector and its numeric score

5. Map groups to components using heading or `<section>` context. If no clear grouping exists,
   use `entry.pageName` as the component name.

6. Produce a page-scoped selector block keyed by `entry.pageName` (see Output Schema below).

**After all pages are processed — merge**

Merge all per-page selector blocks into a single `selectors.json` object. Each top-level key is
a `pageName` (PascalCase). The `_meta` block records all pages processed:

```json
"_meta": {
  "strategy": "dom-based",
  "generatedAt": "...",
  "kitId": "kit-u1",
  "pages": ["AmazonHomePage", "AmazonSearchPage"]
}
```

Write merged `selectors.json` to `outputPath`.
Generate `selectors-lint.html` for any entries with `score < 0.70` across all pages.

---

## Tier 3: AI-Guided Strategy

**Trigger**: `strategy = "ai-guided"` (neither URL nor DOM available)

**Precision goal**: Moderate. Inferred from test case semantics.

### Process

1. Parse `testCases` content — extract all UI element mentions:
   - Look for: input fields, buttons, links, dropdowns, checkboxes, messages, headers
   - Group by page/screen context inferred from test case flow

2. Apply semantic naming conventions to infer likely selectors, then run each through the
   `selector-scorer` skill:
   - "email field" → `getByTestId("email")` → score 0.99 (if testid pattern assumed)
   - "login button" → `getByRole("button", { name: "Login" })` → score 0.95
   - "error message" → `getByRole("alert")` → score 0.92

   **AI-guided note**: Because selectors are inferred (not confirmed against a real DOM), cap
   all scores at **0.75** regardless of attribute type. This flags them for verification without
   blocking generation.

3. Build selectors map (all entries capped at score ≤ 0.75)

4. Write `selectors.json` to `outputPath`

5. Generate `selectors-lint.html` — ALL entries appear since all are inferred

6. **MANDATORY**: Add a `_warning` field to `_meta`:
   ```json
   "_warning": "SELECTORS ARE AI-INFERRED. Verify each selector against the real application before running tests."
   ```

7. Inform the user:
   > "Selectors were inferred from test case descriptions. Please verify `selectors.json` against
   > your application before running tests. Use `/sqe-kit:generate-locators --url <your-app-url>` to
   > replace these with confirmed selectors."

---

## Output Schema (`selectors.json`)

```json
{
  "_meta": {
    "strategy": "playwright-mcp | dom-based | ai-guided",
    "generatedAt": "ISO-8601 timestamp",
    "kitId": "kit-u1",
    "_warning": "Only present when strategy = ai-guided"
  },
  "ComponentName": {
    "locatorAlias": {
      "method": "getByTestId | getByRole | getByLabel | getByPlaceholder | locator",
      "selector": "selector-value",
      "name": "optional role name (for getByRole only)",
      "score": 0.99,
      "reason": "testid | aria-label | placeholder | role | id | text | css | xpath-fallback"
    }
  }
}
```

### Example output — single page
```json
{
  "_meta": {
    "strategy": "playwright-mcp",
    "generatedAt": "2025-10-01T10:00:00Z",
    "kitId": "kit-u1",
    "pages": ["LoginPage"]
  },
  "LoginPage": {
    "emailInput":    { "method": "getByTestId", "selector": "email",    "score": 0.99, "reason": "testid" },
    "passwordInput": { "method": "getByTestId", "selector": "password", "score": 0.99, "reason": "testid" },
    "submitButton":  { "method": "getByRole",   "selector": "button",   "score": 0.95, "reason": "role+name", "name": "Sign in" },
    "errorMessage":  { "method": "getByRole",   "selector": "alert",    "score": 0.92, "reason": "role" }
  }
}
```

### Example output — multi-page (cross-page flow)
```json
{
  "_meta": {
    "strategy": "dom-based",
    "generatedAt": "2026-03-06T00:00:00Z",
    "kitId": "kit-u1",
    "pages": ["AmazonHomePage", "AmazonSearchPage"]
  },
  "AmazonHomePage": {
    "searchInput":  { "method": "getByRole",      "selector": "combobox", "score": 0.95, "reason": "role", "name": "Search Amazon.in" },
    "searchButton": { "method": "getByTestId",    "selector": "nav-search-submit-button", "score": 0.99, "reason": "testid" }
  },
  "AmazonSearchPage": {
    "searchResults":    { "method": "locator",  "selector": "[data-component-type='s-search-result']", "score": 0.88, "reason": "data-attr" },
    "productTitle":     { "method": "locator",  "selector": "h2 a span",                               "score": 0.72, "reason": "css" },
    "resultsHeader":    { "method": "getByRole","selector": "heading",                                  "score": 0.90, "reason": "role" }
  }
}
```

---

## selectors-lint.html

Generate an HTML lint report alongside `selectors.json`. The report must list:
- All selectors with `score < 0.70`
- The component name, locator alias, current selector, current score, and suggested improvement
- A summary count: `X selectors total, Y flagged (score < 0.70)`

If no selectors are below threshold, write: `All selectors passed lint (score ≥ 0.70)`.

---

## Rules for All Tiers

- Component names: `PascalCase` (e.g., `LoginPage`, `CheckoutForm`, `NavigationBar`)
- Locator alias names: `camelCase` (e.g., `submitButton`, `emailInput`)
- Never generate positional selectors (`:nth-child`, `nth-of-type`)
- Never generate absolute XPath — if XPath is unavoidable, use relative XPath only
- If a locator cannot be determined: include it with score ≤ 0.50 and a `"note"` field
- Output file name is always `selectors.json` (never `locators.json`)
