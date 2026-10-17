# Requirement — <Feature name>

Briefed by: <name> · Date: <yyyy-mm-dd>
Screen: <path, e.g. /login>
Figma design: <frame link, if any>

## Dependencies

> What must **exist** before this feature can be tested (not what you do during the test)?
> If nothing, write "None". The "Setup method" column priority order: (1) Page Object of the other
> flow if code exists, (2) API if simple (1–2 calls), (3) full UI if the above aren't viable.
> See `.claude/rules/writing-tests.md` — "Features that depend on data/state..." section.

| Needs | From feature | POM exists | Setup method |
|---|---|---|---|
| Logged in | auth | Yes | `fixtures/auth.fixture.ts` |
| <e.g. at least 1 product in cart> | <e.g. cart> | <Yes/No> | <POM → `pages/.../CartPage.ts`, which method; No → specific API or TBD> |

> **Multi-screen feature** (checkout flow, multi-step registration...) → remove the 2 lines
> "Screen" / "Figma design" above and replace with this table (fill in during the UI survey, step 2):
>
> ## Screens in flow
>
> | Order | Screen | URL | Figma | Direct entry (to test edge cases without navigating from start) |
> |---|---|---|---|---|
> | 1 | <screen name> | <path> | <frame link> | — (first screen in flow) |
> | 2 | <screen name> | <path> | <frame link, may be a different Figma file> | <specific API/storageState, or "TBD" if not yet surveyed> |
>
> The last column is only knowable after surveying the real app — don't guess, don't leave blank,
> write TBD if you haven't found a way yet.

## API triggers outside the UI _(if any)_

> Fill in when the flow needs to call a third-party or internal API to change state that the UI
> can't do (e.g. change shipping status, trigger a webhook, trigger a background job).
> QA pastes curl from devtools/Postman here — `write-test` will convert it to a `request` fixture.

| Step that needs state change | Sample curl | Internal or third-party | Sandbox available | Token fixed or session-based |
|---|---|---|---|---|
| <e.g. change shipping status> | `curl -X POST https://... -d '...'` | <Third-party / Internal> | <Yes / No / Unknown> | <Fixed / Session-based> |

## Input fields

| Field | Type | Required | Limit | Error message |
|---|---|---|---|---|
| <field name> | <text/password/checkbox...> | <Yes/No> | <limit if any> | <error message> |

## Buttons and behaviours

| Button | When enabled | Action |
|---|---|---|
| <button name> | <always / condition> | <what it calls, result on success/failure> |

## Main flow

1. <step 1>
2. <step 2>

## Business rules

- <rule, e.g. wrong X times in a row → lock for Y minutes>

## TBD

- <unresolved question> — TBD

---

## Locators recorded during survey _(technical — step 2)_

> Recorded so `write-test` can reuse them, skipping a second browser survey session.
> **Not guaranteed correct if the UI changes after the survey — verify when coding.**

| Element | Suggested locator | Notes |
|---|---|---|
| <Field/button name> | `getByRole('...')` / `getByLabel('...')` / `getByTestId('...')` | <context, e.g. "only appears when form has errors"> |
