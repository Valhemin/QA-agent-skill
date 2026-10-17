---
name: write-test
description: Use when asked to write a new Playwright test for a feature, convert a manual test case into automation, or add test coverage. Produces Page Object + spec following repo conventions, verified by lint and typecheck.
---

# Writing a new test

Use when QA describes a case to automate, or wants to convert a manual test case into an
automated one.

**Input:** a verbal case description (or an existing manual test case). **Output:**
`pages/<feature>/<Name>Page.ts` + `tests/<feature>/<case-name>.spec.ts`, run green for real ≥1 time.

Read `.claude/rules/writing-tests.md` — naming, tags, code rules, generator/healer agents.

## If the test case was pushed to Google Sheet for QA review — pull before writing

QA says something like "done reviewing, go ahead" / "I've approved it" →

**Step 1**: Stage current changes into the git index first (so the post-pull diff is accurate):
```bash
git add testcases/<system>/<feature>.md
```

**Step 2**: Pull into the md:
```bash
npm run sheet:pull -- --feature=<feature> --sprint=<sprint number used when pushing>
```

**Step 3**: Read the diff to see what QA changed:
```bash
git diff testcases/<system>/<feature>.md
```

Write the test against the diff's content, not the old md from before QA's edits. No need to
ask QA which case changed — git diff already tells you exactly; asking would just waste a
round trip.

## Before writing — mandatory checks

Always open `requirements/<system>/<feature>.md` first, confirm these:

| Question | If unclear |
|---|---|
| Where does this feature sit in the mapping table (`data/feature-map.md`)? | **Ask** — don't create a new folder on your own |
| What's the expected result of the case? | **Ask** — don't define right/wrong yourself |
| Does the test need to be logged in? | Check existing fixtures; not sure → **ask** |
| Where does the test data come from? | Use existing `data/*.json`; need new data → **ask** |
| Is there a "Dependencies" table — does this feature need something set up first? | Yes → follow the 3-tier priority (POM > simple API > full UI) in `writing-tests.md`. Marked TBD → **ask** whether the other feature already has POM |
| Is there a "Screens in flow" table — is this a multi-screen feature? | Yes → write **`main-flow.spec.ts` first** (end-to-end), then per-screen edge cases using the "Direct entry" column. Don't jump straight to edge cases before the happy-path spec exists |
| Did QA send a curl to change state mid-flow (e.g. third-party shipping status change)? | Convert per `.claude/rules/writing-tests.md` → "Mid-test API calls". Don't hardcode a token — ask whether it's dynamic or fixed |

**Never invent a locator.** Before re-opening a browser to look for it, check
`requirements/<system>/<feature>.md` → "Locators recorded during survey" — if already recorded
during `plan-testcases`, reuse it directly, just verify it when running (step 5), no need
for a separate browser session. Not there, or suspected to have changed, two options, don't guess:
- Ask the `playwright-test-generator` agent to open the real app; locators it produces are real.
- Or ask QA for the exact name of the button/field.

## Steps

1. Determine feature → path using `data/feature-map.md`. A new feature that only belongs to
   one system (e.g. only exists on web) → check `playwright.config.ts` already has
   `testIgnore` for the other systems' projects — if missing, add it now, to avoid false
   timeouts when running multiple systems together (see `.claude/rules/writing-tests.md` →
   "testIgnore" section).

2. **Page Object** (`pages/<feature>/<Feature>Page.ts`):
   - Already exists → add any missing locators/methods.
   - Doesn't exist → create it, extending `BasePage`, following the pattern in
     `pages/auth/LoginPage.ts`.
   - Locators are `private readonly`. Public methods named after **business intent**
     (`login()`, `addToCart()`), not the mechanical action (`clickButton2`).
   - Locators used for assertions → expose via a getter (`getErrorMessage()`).
   - **NO `expect()` inside a Page Object.**
   - Locator priority: see `.claude/rules/writing-tests.md` → "Code rules".

3. **Spec** (`tests/<feature>/<feature>-<scenario>.spec.ts`):
   - Import `test`/`expect` from `fixtures/base.fixture` (or `auth.fixture` if login is needed).
   - Test name = behaviour + expected result, e.g. `'show error when password is wrong'`.
   - Add a tag: `@smoke` (main flow, fast) or `@regression` (full coverage).
   - **Add the TC-ID annotation** taken from `testcases/<system>/<feature>.md` (never invent
     a new ID): `{ annotation: [{ type: 'case', description: 'TC-LOGIN-001' }] }` — without
     it the reporter can't back-fill the result into the test case table.
   - Each test is independent, runs in any order. Don't make test A depend on data created
     by test B.
   - Use Playwright's auto-waiting assertions (`expect(locator).toBeVisible()`), **not**
     `waitForTimeout`.

4. **Check — both must be clean:**
   ```bash
   npx tsc --noEmit
   npm run lint
   ```

5. **Run it for real** against the domain (`.claude/rules/running-tests.md`) to confirm the
   test actually runs, not just compiles:
   ```bash
   npx playwright test <spec-path> --project=chromium
   ```
   Fails → read `reports/ai/summary.json`, fix, re-run. **Never hand off a test that hasn't
   passed at least once.**

6. Report: files created/edited, number of tests added, run result, and a "Needs
   clarification" section.

## Feature removed/redesigned — cleaning up old tests

QA says "feature X no longer exists" or "screen Y was completely redesigned":

1. **Don't delete immediately.** Ask: is the feature fully removed, or just the UI changed
   (logic still exists)?
2. Fully removed → delete `tests/<feature>/`, `pages/<feature>/`, remove the feature's row
   from the mapping table (`data/feature-map.md`), delete the matching `testcases/` and
   `requirements/` files (or move to `testcases/_archive/` to keep history — ask QA which
   they prefer).
3. Redesigned (logic stays, UI changes) → **don't delete**, treat it as a
   "requirement changed mid-stream" (README.md → that section), update in place instead of
   rewriting from scratch.
4. Always run `npm run check:docs` after deleting — it catches stale references to deleted files.

## Reference example

Always open `pages/auth/LoginPage.ts` and `tests/auth/login.spec.ts` as a reference before
writing — keep new code consistent with existing code, don't invent your own style.

## Never do this

- **Never invent a locator** and leave QA to fix it later. Better to ask than hand off a
  test that will definitely fail.
- **Never write a test with no assertion** — the `expect-expect` lint rule blocks it.
- Don't combine multiple scenarios into one test. Each test checks one behaviour.
- Never write a test that depends on data assumed to exist in the environment without
  stating it (e.g. test needs order #123 to exist) — add it to "Needs clarification".
- Never use `test.only` (lint blocks it), never leave `test.skip` without a reason.
