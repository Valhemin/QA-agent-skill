---
name: review-edge-cases
description: Use when a tester gives a list of edge cases/test cases for a feature and wants them reviewed, or wants missing edge cases added. Checks against the 7 standard edge-case groups and existing spec files, reports gaps/duplicates without inventing unrelated cases.
---

# Reviewing edge cases

**Input:** the list of edge cases/test cases QA provides. **Output:** text feedback on
gaps/duplicates (no file created) — QA decides whether to add them to `testcases/*.md`.

Read `.claude/rules/edge-cases.md` — the 7 standard edge-case groups.

## Steps

1. Read the list of edge cases the tester provided. No list yet → **ask, don't invent one**.

2. Read `requirements/<system>/<feature>.md` to learn business rules (e.g. character limits,
   required fields) before comparing — avoid suggesting a case that contradicts a recorded
   requirement. Also read `knowledge/<system>/<feature>.md` if it exists — learn whether the
   real feature calls an API that surfaces in UI (affects group 6) and whether it has an
   access-control concept (group 7).

3. Read the existing specs in `tests/<feature>/` to see what's already covered, avoid
   suggesting duplicates. Feature not in the mapping table (`data/feature-map.md`) → ask,
   don't guess the path.

4. Cross-check each of the 7 standard groups against the tester's input:
   - Group already covered → leave it, don't change it.
   - Group missing → suggest a specific case (what input, what expected result).
   - Tester's case falls outside the feature's scope → **ask**, don't drop it silently,
     don't add it silently either.

5. Return a table:
   ```
   | # | Group | Case tester already has | Suggested additional case |
   |---|---|---|---|
   ```
   With a "Needs clarification" section per `.claude/rules/testing-conventions.md` (section 0).

6. Only when the user asks: generate the `.spec.ts` file for a missing case, following
   convention (naming, tags, POM — `.claude/rules/writing-tests.md`).

## Never do this

- Never add edge-case groups beyond the 7 in the rule file. See a risk outside the 7 groups →
  **raise it for a human to decide**, don't add it to the table as a mandatory case.
- Never claim "fully covered" without having read all existing specs — state exactly which
  files you read.
- Never guess the product's business rule. Don't know the correct behaviour → add it to
  "Needs clarification" for the tester to confirm.
