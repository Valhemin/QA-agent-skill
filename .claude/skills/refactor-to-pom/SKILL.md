---
name: refactor-to-pom
description: Use after playwright-test-generator produces a raw spec, when a spec contains inline locators, or when lint reports no-raw-locators errors. Moves locators out of the spec into a Page Object so the spec only calls methods and asserts.
---

# Converting a raw spec to Page Object Model

The Playwright `playwright-test-generator` agent generates inline locator calls
(`page.getByRole(...).click()`) directly in the spec. This repo uses Page Objects. This
skill bridges the two.

**Input:** a raw spec with locators inline. **Output:**
`pages/<feature>/<Name>Page.ts` (locators + methods) + the spec rewritten to call only
methods and assertions.

Read `.claude/rules/writing-tests.md` — code rules, generator agent section.

## Steps

1. Read `knowledge/<system>/<feature>.md` if it exists — learn the URL and any state quirks
   before verifying locators after the refactor. Then read the raw spec to convert. Identify
   its feature using the mapping table in `data/feature-map.md`.
   Feature not in the table → **stop, ask the user**, don't create a new folder on your own.

2. Find the matching Page Object at `pages/<feature>/<Feature>Page.ts`:
   - Already exists → add any missing locators/methods to that class.
   - Doesn't exist → create it, extending `BasePage`, following the pattern in
     `pages/auth/LoginPage.ts`.

3. For each locator in the raw spec:
   - Move it to a `private readonly` field on the Page Object.
   - Group consecutive actions on the same screen into **one public method named after the
     business intent** (`login(email, password)`), not the mechanical action (`clickButton3`).
   - Locators used for assertions → expose via a getter (`getErrorMessage()`), **never** put
     `expect()` inside the Page Object.

4. Rewrite the spec:
   - Import `test`/`expect` from `fixtures/base.fixture`.
   - Test body should only contain: Page Object method calls + `expect(...)`.
   - Keep the test names and business steps the generator verified — **don't change the
     assertion intent**, don't relax conditions to make the test pass more easily.
   - Add the `@smoke` or `@regression` tag per `.claude/rules/writing-tests.md`.

5. Check — both must be clean:
   ```bash
   npx tsc --noEmit
   npm run lint
   ```
   The `no-raw-locators` lint rule will report any raw locators still left in the spec.

6. **Run the refactored spec** to confirm it still passes — tsc/lint don't catch logic
   errors (wrong method name, wrong argument when collapsing locators):
   ```bash
   npx playwright test <spec-path> --project=chromium
   ```
   Fails after the refactor → tell the user, **don't silently weaken the assertion to make
   it pass** (see "Never fake a passing test" in `writing-tests.md`).

7. Report: list the files changed, the new methods added to the Page Object, and a "Needs
   clarification" section per `.claude/rules/testing-conventions.md` (section 0).

## Never do this

- Never change an assertion to make the test pass. Test fails after refactor → report to the
  user, don't fix it silently.
- Never merge multiple features into one Page Object.
- Never delete a test the generator produced because it "looks redundant" — ask first.
