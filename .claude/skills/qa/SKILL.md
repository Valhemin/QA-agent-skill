---
name: qa
description: Use for ANY QA automation request in this repo — running tests, writing tests, planning test cases, reviewing edge cases, checking UI against Figma, reporting bugs. Single entry point; routes to the right workflow. Trigger on any mention of test, run test, write test, regression, test case, bug, Figma, edge case.
---

# QA — single entry point

The user speaks in plain language. Detect intent → read the matching skill → act.

## Routing

| User says | Read next |
|---|---|
| "run test X", "regression", "any bugs" (wants to **run** a test) | `.claude/skills/run-tests/SKILL.md` |
| "ready to release", "summarize test cases", "anything before release" (wants a **summary**, not a new run) | `.claude/skills/release-readiness/SKILL.md` |
| "write a test for X" (case already clear) | `.claude/skills/write-test/SKILL.md` |
| Long requirement brief, "plan the test" | `.claude/skills/plan-testcases/SKILL.md` |
| Pastes a list of cases, "what's missing" | `.claude/skills/review-edge-cases/SKILL.md` |
| Has a Figma link, "compare with design" | `.claude/skills/figma-ui-check/SKILL.md` |
| "write up a bug", "send to dev" | `.claude/skills/report-bug/SKILL.md` |
| Spec has raw locators that need standardising | `.claude/skills/refactor-to-pom/SKILL.md` |
| "clean up test data", "any leftover test data", `leftover_data` is non-empty | `.claude/skills/cleanup-testdata/SKILL.md` |
| "requirement just changed", "brief changed", "feature removed/redesigned" | **README.md → "Requirement changed mid-brief"** — update requirements + testcases + code in that exact order |

Not sure between two branches → ask. Multiple requests bundled → handle sequentially, report once at the end.

## Mandatory rules

**Never guess.** Unclear → ask. No data → say "no data yet", don't invent locators/results.
Something unclear → skip it, add it to "Needs clarification" at the end, keep working on the rest.

**Read memory first, write after.** Before acting: read `knowledge/<system>.md` (common)
then `knowledge/<system>/<feature>.md` (feature-specific) if it exists — systems declared in
`data/he-thong.json`. Learned something new (failed then succeeded, a hidden flow, a business
rule QA stated) → record it immediately with a date. Only record what's verified.

**Check the repo before asking**: `knowledge/` (how it actually works) ·
`requirements/<system>/<feature>.md` (how it should work) ·
`testcases/<system>/<feature>.md` (cases + results) · `reports/ai/summary.json` (last run) ·
`pages/auth/LoginPage.ts` (sample code). Only ask when you can't find it, batch into one message.

**After editing test code**: `npx tsc --noEmit` and `npm run lint` must be clean. New tests
must pass for real at least once. Haven't run it → say so.

**Never fake a passing test**: never change an expected value to match a broken app, never
use `test.fixme`/`toBeTruthy`/`force: true` to get through CI. `npm run check:tests` blocks
this. Genuinely necessary → add `// Reason: ...` on the line above. The `healer` agent makes
this mistake often — read every line it changes carefully.

**Production**: ask for confirmation before running, tests can create real data.

## Saving tokens

Read `reports/ai/summary.json`, **don't** read terminal logs / HTML report / `trace.zip`
(binary). Run tests with `> /dev/null 2>&1`. Investigating a failure: at most **2 files**
in `reports/qa/diagnostics/<feature>/`.

## Reporting

Max 15 lines. Group failures with the same root cause into one line. Business language, no
stack traces. Use the `next_steps` already in `summary.json`.

Two fields to watch: `knowledge_warnings` (memory may be stale → re-read that file, fix
immediately) · `leftover_data` (leftover test data → list it, **ask** before deleting, only
delete once confirmed).

End with:
```
## Needs clarification
- <unclear item> — <what's missing>
```
Nothing unclear → write "None".

## Details when needed

| Task | Read |
|---|---|
| Full ruleset (naming, tags, 7 edge-case groups, 6-step process) | `.claude/rules/testing-conventions.md` |
| Test accounts, performance/a11y checks, cleaning up leftover data | `.claude/skills/qa/chi-tiet.md` |
