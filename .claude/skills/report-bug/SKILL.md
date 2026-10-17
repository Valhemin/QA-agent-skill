---
name: report-bug
description: Use after a test failure is confirmed to be a real bug, OR when QA finds a bug through manual testing, and needs a ready-to-paste bug ticket with reproduction steps, evidence, and FE/BE assignment suggestion.
---

# Writing a bug ticket for dev

Use after a test failure is confirmed to be a real bug (not a stale test or an environment
issue), OR when QA finds a bug through manual testing. Goal: produce content ready to paste
directly into a ticket so dev can reproduce it immediately.

**Input:** `reports/ai/summary.json` + `reports/qa/diagnostics/` from the confirmed-bug run.
**Output:** bug ticket as plain text following the template below (no file created, paste
straight into the ticket).

Read `.claude/rules/running-tests.md` — leftover data / output section.

## Steps

1. Read `reports/ai/summary.json`, find the failing test to report. Pull out: `repro_steps`,
   `triage`, `error`, `screenshot`, `diagnostics`.

2. Read the matching `reports/qa/diagnostics/<name>.md` to see what the screen actually
   showed and which requests errored.

3. **Before writing the ticket, check the precondition**: `triage.verifyFirst` states what QA
   needs to verify manually. If the user hasn't done that yet → **ask whether they've already
   checked it by hand**. Not confirmed as a real bug yet → don't write the ticket; avoid
   wasting dev time on a false positive.

4. Choose a **severity** from the table below, based on the affected flow (not on the
   technical error category — category says "what kind of error", severity says "how wide the
   impact is").

   | Severity | When | Example |
   |---|---|---|
   | Blocker | Main flow completely unusable, no workaround | Can't tap the Checkout button |
   | Critical | Wrong money/data, or a security exposure | Price displayed wrong after discount |
   | Major | Feature broken but there's an alternative path | Product filter wrong, can still search |
   | Minor | Visual misalignment, doesn't block action | 4px margin off on mobile |

   This is the **AI's suggestion** — state the reason for the choice. QA makes the final
   call and may change it.

5. Write the ticket following the `.claude/templates/bug-report.example.md` template. Any
   data not actually available → write "unknown", **never invent it**.

## After dev says the fix is in

Re-run exactly that test/case (`npx playwright test --grep "<feature>"` or by the `TC-xxx`
code in `testcases/`). The test case table **back-fills Pass/Fail automatically** — no
manual status update needed. Actually passes → tell QA to close the ticket. Still fails →
report back to dev with the new run's result, don't assume "probably not merged yet".

## Rules

- The title must describe the **symptom**, not the test name. Write "Price after discount
  displays wrong", not "test cart.spec.ts fail".
- Reproduction steps must be complete enough for someone with no prior knowledge to follow.
  If the login step or product-selection step is missing, add it based on the fixture the
  test uses.
- `triage.owner` is a **suggestion**, not a verdict. Always state the confidence level, and
  for `unknown` say explicitly what needs to be verified before assigning.
- Never combine multiple distinct bugs into one ticket. One symptom = one ticket.
- Bug only on one screen size → include that size in the title, e.g. `[Mobile 375px]`.

## Never do this

- Never write the ticket when `triage.owner` is `QA` and the error group is `environment` —
  that's a QA-internal fix, not a product bug.
- Never state the root cause in the product code (e.g. "caused by calculatePrice being
  wrong") when you can only see it from the outside. Describe the symptom and the evidence,
  let dev find the root cause.
