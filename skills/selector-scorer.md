# Selector Scorer — Reusable Skill

Provides deterministic numeric scoring for UI element selector candidates. Replaces the
string-based `"confidence": "confirmed|likely|inferred"` system with precise numeric scores.
Load this skill into the locator-agent whenever selector candidates need to be ranked.

---

## Scoring Function

```typescript
export interface ScoredSelector {
  selector: string;
  score: number;   // 0.0 → 1.0 (higher = more stable)
  reason: string;  // human-readable explanation
}

export function scoreSelector(el: ElementCandidate): ScoredSelector {
  // Priority 1 — Explicit test ID attribute (purpose-built for testing)
  if (el['data-testid']) return {
    selector: `getByTestId("${el['data-testid']}")`,
    score: 0.99, reason: 'data-testid'
  };

  // Priority 2 — Other purpose-built test attributes
  if (el['data-cy'])   return { selector: `locator('[data-cy="${el['data-cy']}"]')`,   score: 0.97, reason: 'data-cy' };
  if (el['data-test']) return { selector: `locator('[data-test="${el['data-test']}"]')`, score: 0.97, reason: 'data-test' };
  if (el['data-qa'])   return { selector: `locator('[data-qa="${el['data-qa']}"]')`,     score: 0.96, reason: 'data-qa' };

  // Priority 3 — ARIA label (semantic + accessible)
  if (el['aria-label']) return {
    selector: `getByLabel("${el['aria-label']}")`,
    score: 0.92, reason: 'aria-label'
  };

  // Priority 4 — Placeholder text (stable for form inputs)
  if (el['placeholder']) return {
    selector: `getByPlaceholder("${el['placeholder']}")`,
    score: 0.88, reason: 'placeholder'
  };

  // Priority 5 — ARIA role (requires name to be most useful)
  if (el['role'] && el['accessibleName']) return {
    selector: `getByRole('${el['role']}', { name: '${el['accessibleName']}' })`,
    score: 0.85, reason: 'role+name'
  };
  if (el['role']) return {
    selector: `getByRole('${el['role']}')`,
    score: 0.78, reason: 'role'
  };

  // Priority 6 — Form name attribute
  if (el['name']) return {
    selector: `locator('[name="${el['name']}"]')`,
    score: 0.75, reason: 'name'
  };

  // Priority 7 — Stable CSS id (warn if looks auto-generated)
  if (el['id']) {
    const isDynamic = /ember\d+|react-\d+|css-[a-z0-9]+|\d{5,}/.test(el['id']);
    return {
      selector: `locator('#${el['id']}')`,
      score: isDynamic ? 0.40 : 0.72,
      reason: isDynamic ? 'id-dynamic-WARNING' : 'id'
    };
  }

  // Priority 8 — Visible text (fragile to copy changes)
  if (el['text']) return {
    selector: `getByText("${el['text']}", { exact: true })`,
    score: 0.65, reason: 'text'
  };

  // Priority 9 — CSS selector (moderate stability if not auto-generated)
  if (el['css']) {
    const isGenerated = /css-[a-z0-9]{5,}|__[a-z]+_[a-z0-9]+/.test(el['css']);
    return {
      selector: `locator('${el['css']}')`,
      score: isGenerated ? 0.35 : 0.60,
      reason: isGenerated ? 'css-generated-WARNING' : 'css'
    };
  }

  // Priority 10 — XPath (last resort)
  return {
    selector: `locator('xpath=${el['xpath']}')`,
    score: 0.50, reason: 'xpath-fallback'
  };
}
```

---

## Score Thresholds

| Score Range | Status | Action |
|---|---|---|
| 0.90 – 1.00 | ✅ Excellent | Use as-is |
| 0.70 – 0.89 | ✓ Good | Use with no warning |
| 0.50 – 0.69 | ⚠️ Fair | Flag in selectors-lint.html; user should verify |
| 0.00 – 0.49 | ❌ Poor | Flag prominently; require manual review before running tests |

---

## Selector Lint Report (`selectors-lint.html`)

Generated alongside `selectors.json` by locator-agent when any selector has `score < 0.70`.
Content: table of flagged selectors with score, reason, recommended fix.

**Common lint rules:**
- `score < 0.70`: flagged for review
- `reason = "id-dynamic-WARNING"`: "Replace with `data-testid` or ARIA attribute"
- `reason = "css-generated-WARNING"`: "CSS-in-JS generated class — extremely fragile"
- XPath depth > 3 (count `/` separators): "Simplify XPath or add `data-testid` to element"
- Any selector containing `nth-child` or `nth-of-type`: "Positional selector — will break on DOM reorder"
- `score = 0.50` (xpath-fallback): "Add `data-testid` attribute to the element in the application"

---

## Usage in locator-agent

```
For each element candidate discovered (via MCP or DOM parser):
  scored = scoreSelector(element)
  Include scored.selector, scored.score, scored.reason in selectors.json output
  If scored.score < 0.70: add to lint report
```

The locator-agent should NEVER output a selector without running it through this scorer first.
