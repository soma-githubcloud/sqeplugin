# User Story Quality Rules

Apply these rules when generating or reviewing user story files (`US-NNN-*.md`).
These are enforced by `user-story-writer.agent.md` before saving any story file.

---

## Structure Rules (Non-Negotiable)

1. **Every story must have all four sections**: User Story, Acceptance Criteria, Technical Context, Notes
2. **US ID format**: `US-NNN` (zero-padded to 3 digits minimum, e.g., `US-001`, `US-042`)
3. **Source traceability**: every story must reference the source file and feature ID in the header
4. **Status field**: always set to `Draft` on generation; reviewers change it to `Reviewed` or `Approved`

---

## "As a / I want / So that" Rules

5. **Role specificity**: never use `"a user"` if a more specific role is identifiable from the source.
   Acceptable roles: `Patient`, `Clinician`, `Administrator`, `Guest`, `Registered User`, `API Consumer`, etc.

6. **Goal is actor-centric**: `"I want to"` must describe what the **actor** does or achieves — not what the system does internally.
   - ❌ `I want the system to validate the form`
   - ✅ `I want to submit a form and receive immediate feedback`

7. **Benefit is user value**: `"So that"` must describe an outcome meaningful to the actor — not a technical state.
   - ❌ `So that the record is persisted to the database`
   - ✅ `So that my progress is saved and I can continue later`

8. **One goal per story**: if the story requires two unrelated `I want` statements, split into two stories.

---

## Acceptance Criteria Rules

9. **Minimum 3 ACs per story**: if fewer than 3 can be derived from the source, mark extras as `[TO BE CONFIRMED]`

10. **Maximum 10 ACs per story**: if more than 10 are needed, consider splitting the story

11. **Each AC is singular**: one assertion per AC — never compound with `and`
    - ❌ `The form submits and the user sees a confirmation message`
    - ✅ `AC-01: The form submits successfully when all required fields are populated`
    - ✅ `AC-02: A confirmation message "Submission received" is displayed after successful submission`

12. **ACs are specific and measurable**: no vague qualifiers
    - ❌ `The page loads quickly`
    - ✅ `The page renders within 2 seconds under normal network conditions`
    - ❌ `An error is shown`
    - ✅ `An error message "Email address is required" is displayed below the email field`

13. **ACs are written in present tense**: describe the system state, not a test action
    - ❌ `User clicks the button and sees a result`
    - ✅ `When the user submits the form, a success banner is displayed`

14. **AC coverage minimum**: generated ACs must cover at minimum:
    - At least one happy path (primary success scenario)
    - At least one validation rule (if applicable)
    - At least one error/edge condition (if inferable from source)

15. **AC IDs are sequential**: `AC-01`, `AC-02`, etc. — never restart numbering within a story

---

## Technical Context Rules

16. **No blank fields**: every row in Technical Context must have a value or `N/A`

17. **NFRs are specific**: if a performance SLA is listed, include the actual number
    - ❌ `Response time: fast`
    - ✅ `Response time: < 2 seconds (95th percentile)`

18. **Compliance rules are traceable**: reference the specific regulation or standard
    - ❌ `Security: required`
    - ✅ `Security / Compliance: HIPAA §164.312 — access controls required`

---

## Notes Section Rules

19. **Open Questions are genuine unknowns**: do not include questions whose answers are already in the source document

20. **Constraints are from source only**: do not invent constraints — only include what is stated or clearly implied in the requirement document

21. **If no open questions**: omit the Open Questions subsection entirely (do not write an empty section)

---

## Traceability Rules

22. **Source section is required**: if the source document has sections, reference the specific section where this feature was found

23. **Feature ID is preserved**: `F-NNN` from `feature-map.json` must appear in every story header — enables round-trip from story back to source requirement

24. **US-INDEX.md must be complete**: every generated story must appear in the index; no orphan files

---

## Anti-Patterns Reference

| Anti-Pattern | Rule Violated | Fix |
|---|---|---|
| `As a user, I want...` (no specific role) | Rule 5 | Use specific role from source |
| `So that the API returns 200` | Rule 7 | Rewrite as user value |
| AC with "and" combining two assertions | Rule 11 | Split into two ACs |
| AC: "The form works correctly" | Rule 12 | Specify what "correctly" means |
| Missing Technical Context row | Rule 16 | Fill with N/A |
| Open question already answered in source | Rule 19 | Remove the question |
| Story with 8+ ACs covering 3 different workflows | Rule 10 + Rule 8 | Split the story |
| `[TO BE CONFIRMED]` on more than 2 ACs | — | Indicates very sparse source; flag for stakeholder review |
