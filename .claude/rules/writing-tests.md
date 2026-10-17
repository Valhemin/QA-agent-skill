# Writing tests — code rules, naming, tags, agents

Read when: generating test code (`write-test`), converting raw specs to POM (`refactor-to-pom`).

## Feature ↔ path ↔ tag mapping (must stay in sync)

Each feature = one subfolder under `tests/<feature>/` and `pages/<feature>/`, with tag `@<feature>`.
Infer path/tag from the feature name the tester gives — don't ask again if the mapping is clear.
Feature NOT in the table → ask, don't guess the path.

**The mapping table lives in `data/feature-map.md`** (extracted from the rule file — this table
changes when features are added; the rule file should stay stable. Separating them means adding
a feature only touches one data file, reducing git conflicts when two QAs add features at once).

## Features that depend on data/state from another feature

Read the "Dependencies" section in `requirements/<system>/<feature>.md` before writing.

- **Needs login** → import `test`/`expect` from `fixtures/auth.fixture.ts` instead of
  `fixtures/base.fixture.ts`. Already exists, don't re-implement.

- **Needs data from another feature** (item in cart, existing order...) → use
  `test.beforeAll` running **once** for the whole describe block, in priority order:

  **Priority 1 — Reuse the existing flow's Page Object.** If the other flow already has POM
  code, reuse it directly without knowing the API internals:

  ```ts
  test.describe('Shipping address — edge case @checkout', () => {
    test.beforeAll(async ({ browser }) => {
      const ctx = await browser.newContext();
      const page = await ctx.newPage();
      const cartPage = new CartPage(page); // already-tested POM from the prior flow
      await cartPage.addProduct('sp-001');
      await cartPage.proceedToCheckout();  // navigate to the screen under test, persist state
      await ctx.close();
    });

    test('missing house number shows error @regression', async ({ page }) => {
      await page.goto('/checkout/address'); // state already set up in beforeAll
      // test the actual edge case
    });
  });
  ```

  Benefit: no API dependency; if the setup flow changes the code fails immediately, no
  separate maintenance needed.

  **Priority 2 — Call the API** only when the setup flow has no code yet **and** the API
  is simple (1–2 calls, no need to combine many endpoints). Use Playwright's built-in
  `request` fixture:

  ```ts
  test.beforeAll(async ({ request }) => {
    const res = await request.post('<API noted in requirements>', { data: {/* ... */} });
    idData = (await res.json()).id;
  });
  ```

  Avoid API calls when you need to combine many calls to build data — that's as complex as
  backend code, hard to maintain, and not QA's job.

  **Priority 3 — Fallback:** no POM and a complex API → run the full UI in `beforeAll`
  (still better than `beforeEach` since it only runs once), accept that it's slower.

  `beforeAll` not `beforeEach` — many edge cases share one data setup, create once for the
  whole describe, avoids environment clutter and slowness.

  **Warning: any case that cancels/edits/deletes shared state must go in a separate describe.**
  If one case (e.g. "cancel order succeeds") destroys the order that `beforeAll` created,
  later cases in the same describe will fail for lack of data — not a bug, just execution
  order breaking things. Indicator: the case description contains a verb that destroys the
  object `beforeAll` just created. Fix: move that case to its own `test.describe` with its
  own `beforeAll` that builds its own state.

- Requirements say TBD here → **ask: does the other feature already have POM?** Yes →
  use Priority 1. No → ask whether to use the API or go full UI.

## Time-dependent test cases (promotions, flash sales, expired codes...)

Never wait for real time to pass. Playwright's `page.clock` controls fake browser time —
use it when a feature has logic like "expires at X", "starts after Y hours".

```ts
test('promo code shows expired after 23:59 on end date @regression', async ({ page }) => {
  // Set the browser clock to the exact moment under test — BEFORE goto
  await page.clock.setFixedTime(new Date('2025-12-31T23:59:30'));

  await page.goto('/cart');
  await cartPage.applyPromoCode('SALE50');

  // After "midnight" passes, advance time:
  await page.clock.fastForward('00:01:00'); // step 1 minute forward
  await expect(page.getByText('Promo code has expired')).toBeVisible();
});
```

| Need | API |
|---|---|
| Fix the clock at a specific moment | `await page.clock.setFixedTime(new Date('...'))`  |
| Step time forward after setting | `await page.clock.fastForward('hh:mm:ss')` |
| Clock runs from a fake start time | `await page.clock.install({ time: ... })` then let it run |

**Browser-only** — the `request` fixture and the server are not affected. If the feature
validates time server-side (token expiry checked by the backend, not a client-side JS timer)
→ `page.clock` won't help; use a mid-test API call to set the expired state from the
backend, or mark the case `Automation: ❌`.

Record in `requirements/<feature>.md` → "Business rules": whether time is evaluated
client-side or server-side. `plan-testcases` step 1 must ask if the feature has time-based rules.

## File upload (product images, avatars, CSV import...)

Use `page.getByLabel('...').setInputFiles(path)` or `waitForEvent('filechooser')` when the
input is hidden. Sample files go in `data/files/`, committed to the repo (valid, too-large,
wrong-format, import CSV).
Verify results: don't only check a success toast — verify the real image/URL appears, or call
the API and check for 200.

## OTP / Email verification

Keep `Automation: ❌`. Add to the Notes column: `manual — needs real email`. Don't
generate fake OTPs.

## Mid-test API calls to change state quickly

Use when a step in the middle of a flow needs to change state via a third party/webhook
(can't click through UI). The API call goes inside `test()`, not in `beforeAll` — it's a
test step, not setup.

Pattern: UI reaches the required state → `request.post(url, { headers, data })` → UI verifies
the reaction.

**Keys/tokens**: don't hardcode, don't use `.env`. Store in `data/api-keys.json` (gitignored).
Short-lived session tokens (JWT) → fetch dynamically after login, don't store to file.

**Ask before converting a curl QA sends**: internal API or third party? Does the dev
environment have this endpoint? Is the key fixed or rotated? Does UI reflect immediately or
with a delay?

**Don't use this pattern when**: state requires 3+ API calls to build (→ use POM/full UI),
or the third-party API has no sandbox.

## Multi-screen features (checkout flow, multi-step registration...)

When `requirements/<system>/<feature>.md` has a "Screens in flow" table (see
`.claude/templates/requirement.example.md`), write tests in this priority order:

1. **One happy-path spec**, going end-to-end from the first screen to the last —
   `tests/<feature>/luong-chinh.spec.ts`, tag `@<feature>`. This is the most valuable test:
   it confirms the screens connect correctly, not just that each screen works in isolation.
2. **Per-screen edge cases, written after** — one file per screen
   (`tests/<feature>/<screen-name>.spec.ts`, Page Object `pages/<feature>/<ScreenName>Page.ts`).
   **Don't navigate from screen 1 for every case** — it's slow, and the more steps you pass
   through the more likely to go flaky on a step unrelated to the case being tested.

   Use the "Direct entry" column recorded in requirements (from the UI survey) to set up
   state and `page.goto()` directly:

   ```ts
   test.beforeEach(async ({ request, page }) => {
     // Setup from "Direct entry" column in requirements — varies per feature,
     // no universal formula.
     await request.post('<API noted in requirements>', { data: { /* ... */ } });
     await page.goto('/screen-under-test');
   });
   ```

   Column says TBD or isn't in requirements yet → **ask how to navigate directly to that
   screen** before writing; don't silently loop back from screen 1 "to be safe".

Adding a new feature: add exactly one row to the mapping table + create exactly two folders.

## Feature that only belongs to one system → add `testIgnore` for the other projects

Lesson 2026-10-15: `@auth` tests ran on the `admin`/`affiliate` projects (neither has this
login form) and caused 28 false timeouts — not a bug, just running against the wrong scope.

When a new feature only exists in one system (e.g. `@auth` is web-only), add `testIgnore`
for the other systems' projects in `playwright.config.ts`:

```ts
{ name: 'admin', use: {...}, testIgnore: ['**/auth/**'] },
```

After adding a new feature, verify: `npx playwright test --project=<other-system>` must
not pull in the new feature's tests.

## `getByText(..., { exact: true })` — match the exact displayed text, including punctuation

Lesson 2026-10-15: a locator typed from visual memory failed because the real text in the
app had a trailing punctuation mark. `exact: true` does a full-string match so one character
off is a failure — not an app bug, just a wrong locator.

Before finalising an `exact: true` locator for text that could be ambiguous (punctuation,
whitespace, capitalisation): confirm the real text with `browser_find` or `browser_evaluate`
reading `textContent` directly from the DOM — don't retype from memory or from a screenshot.

## File naming — standard ruleset (applies to every feature, scales from 1 screen to many)

General principle: **don't repeat information that the file's location already conveys**.
A spec inside `tests/<feature>/` doesn't need `<feature>` in its name — the screen/behaviour
name is enough.

### Root folder structure

```
tests/<feature>/          ← spec files
pages/
  <feature>/              ← Page Objects per feature
  shared/                 ← Components reused across features (Header, Modal...)
  BasePage.ts             ← Base class; every <Feature>Page extends this
api/                      ← Typed API clients (when one endpoint is used in many places)
fixtures/                 ← Files that call test.extend() only
utils/                    ← Plain helper functions (not through the test object)
data/                     ← JSON config + mapping
```

### Single-screen feature (e.g. login, add address)

| Type | Convention | Example |
|---|---|---|
| Test spec | `<behaviour>.spec.ts`, kebab-case, describes the behaviour | `tests/auth/login-wrong-password.spec.ts` |
| Page Object | `<Feature>Page.ts`, PascalCase, extends `BasePage` | `pages/auth/LoginPage.ts` |
| Base Page (shared methods) | `BasePage.ts`, PascalCase, in `pages/` | `pages/BasePage.ts` |
| Shared component (cross-feature UI) | `<Name>Component.ts`, PascalCase, in `pages/shared/` | `pages/shared/HeaderComponent.ts` |
| API client (typed HTTP wrapper) | `<name>-client.ts`, kebab-case, in `api/` | `api/auth-client.ts` |
| Fixture (extends Playwright `test`) | `<name>.fixture.ts`, kebab-case, in `fixtures/` | `fixtures/auth.fixture.ts` |
| Util/helper (direct import, no `test.extend`) | `<name>.ts`, kebab-case, in `utils/` | `utils/users.ts`, `utils/quality-check.ts` |
| Runtime data (URLs, accounts, keys) | kebab-case `.json`, in `data/` | `data/test-users.json` |
| Structural data (mapping, not runtime config) | kebab-case `.md`, in `data/` | `data/feature-map.md` |

**`BasePage`** holds shared helpers for every screen: wait for overlay to disappear, scroll
into view, dismiss toasts/popups... Every `<Feature>Page` extends `BasePage` instead of
copying these methods. Only create `BasePage.ts` once ≥2 Page Objects need the same helper —
don't pre-create it "just in case".

**`pages/shared/<Name>Component.ts`** for widgets that appear in ≥2 features (Header with
cart count, Footer, Toast, Confirm Modal...). A `<Feature>Page` composes it via
`new HeaderComponent(this.page)` instead of copying the locators into each Page Object.
Only create when the widget genuinely appears in ≥2 places.

**`api/<name>-client.ts`** for APIs called repeatedly across tests/specs — class wraps
`APIRequestContext`, methods return typed objects, **no assertions**. See `api/README.md` for
how to write and wire into a fixture. Only create when one endpoint is called ≥2 times in
the repo — a single mid-flow call is fine as `request.post(...)` inline in the spec (see
"Mid-test API calls" above), no client class needed.

**Fixture vs util**: a fixture extends Playwright's `test` object via `test.extend({...})`;
the spec imports `test` from it (`fixtures/auth.fixture.ts` → spec gets `{ loginPage }` ready),
placed in `fixtures/`. A util is a direct-import function; the spec does
`import { getUser } from '../utils/users'`, placed in `utils/`.

### Multi-screen feature (e.g. checkout: cart → address → payment → confirmation)

| Type | Convention | Example |
|---|---|---|
| Happy-path spec (end-to-end) | `main-flow.spec.ts` (fixed name — see "Multi-screen features") | `tests/checkout/main-flow.spec.ts` |
| Per-screen edge case spec | `<screen-name>.spec.ts` — **no** feature prefix | `tests/checkout/address.spec.ts` |
| Specific-situation edge case | `<screen-name>-<situation>.spec.ts` | `tests/checkout/address-missing-house-number.spec.ts` |
| Per-screen Page Object | `<ScreenName>Page.ts` — **no** feature prefix | `pages/checkout/AddressPage.ts` |
| Orchestrator Page Object (≥3 screens) | `<Feature>Flow.ts` — aggregates `<ScreenName>Page` children, provides shortcut methods for `beforeAll`/`beforeEach` | `pages/checkout/CheckoutFlow.ts` |

**`<Feature>Flow.ts`** contains only:
- `public readonly` fields pointing to each child `<ScreenName>Page` (initialised in the constructor).
- Multi-step orchestration methods (e.g. `goToPayment()` navigates from cart to the
  payment screen) — use when many specs need to stop at the same mid-flow point.
- **No** private locators, **no** `expect()` — locators always belong to `<ScreenName>Page`.

```ts
// pages/checkout/CheckoutFlow.ts
export class CheckoutFlow {
  readonly cart: CartPage;
  readonly address: AddressPage;
  readonly payment: PaymentPage;

  constructor(page: Page) {
    this.cart = new CartPage(page);
    this.address = new AddressPage(page);
    this.payment = new PaymentPage(page);
  }

  /** Add a product and navigate to the payment screen — used in beforeAll for payment-screen edge cases. */
  async goToPayment(productId: string) {
    await this.cart.addProduct(productId);
    await this.cart.proceedToCheckout();
    await this.address.fillAddress(/* default valid address */);
    await this.address.proceedToPayment();
  }
}
```

Only create `<Feature>Flow.ts` when the feature has ≥3 connected screens. For a single
screen or a simple two-screen flow, instantiate each `<ScreenName>Page` directly in the spec.

## Tags

Each test has **exactly one** of these three tag types:

| Tag | When it runs | Notes |
|---|---|---|
| `@smoke` | Every PR | Main flow, must be fast |
| `@regression` | Nightly + on tester request | Full coverage, includes edge cases |
| `@seed` | **Never runs in the suite** | Reserved for `tests/seed.example.spec.ts` — a style example for the generator agent |

A test can have both `@smoke` and `@regression`. NEVER put `@seed` on a real business test.

`tests/seed.example.spec.ts` lives at the root of `tests/` (not in a subfolder) — the only
intentional exception: a style example for the generator agent, not tied to any feature.
Don't create other files at the `tests/` root.

## Code rules

- Page Object: locators are `private readonly`, every public method = one action, **NO `expect()` inside a Page Object**.
- Assertions go in spec files only.
- Specs import `test`/`expect` from `fixtures/base.fixture.ts`, **never** directly from `@playwright/test`.
- Test title = behaviour + expected result, in plain English. Don't use `test('test 1')`.
- No hard `page.waitForTimeout()` — use Playwright's auto-wait (`expect(locator).toBeVisible()`).
- Don't use `networkidle`.
- **Locator priority (single source of truth):** `getByRole` → `getByLabel` → `getByTestId` → `getByText`. CSS selectors only when nothing else works — the `no-raw-locators` lint rule blocks raw locators in specs.

### Test case IDs

Format: `TC-<FEATURE>-<3-digit number>` for single-screen features,
`TC-<FEATURE>-<SUBFLOW>-<3-digit number>` for multi-screen features.

| Feature type | Format | Example |
|---|---|---|
| Single screen | `TC-<FEATURE>-<001>` | `TC-LOGIN-001` |
| Multi-screen — happy path | `TC-<FEATURE>-<001>` (no subflow needed) | `TC-CHECKOUT-001` |
| Multi-screen — per-screen edge case | `TC-<FEATURE>-<SUBFLOW>-<001>` | `TC-CHECKOUT-ADDR-001` |

**SUBFLOW rule**: 3–6 uppercase characters describing the screen, declared at the top of
`testcases/<system>/<feature>.md` so the whole team uses the same abbreviations. Checkout example:

| Screen | SUBFLOW | ID range |
|---|---|---|
| Cart | `CART` | `TC-CHECKOUT-CART-001..099` |
| Address | `ADDR` | `TC-CHECKOUT-ADDR-001..099` |
| Payment | `PAY` | `TC-CHECKOUT-PAY-001..099` |
| Confirmation | `CONFIRM` | `TC-CHECKOUT-CONFIRM-001..099` |

Once assigned, an ID **never changes and is never reused**, even when a case is deleted —
prevents history confusion.

### Column `Steps` — **must be filled; never leave blank**

The `Steps` column **must have content for every case** — including `Automation: ❌` or `⏸️`.
It's what QA and devs read to reproduce a case without opening code. When creating or updating
a test case table, check this column before finishing — any blank cell means the work isn't done.

In the md file, each step uses the prefix `B<n>:` and is separated by `<br>` (markdown tables
don't support real newlines in cells). When pushed to Google Sheet, `<br>` converts to real
line breaks:

```
B1: Go to /login<br>B2: Enter valid email<br>B3: Enter wrong password<br>B4: Click "Sign in"
```

Displayed on Sheet after push:
```
B1: Go to /login
B2: Enter valid email
B3: Enter wrong password
B4: Click "Sign in"
```

Don't write vaguely ("perform login") — write each click/input step clearly. This column
is what QA reads on Google Sheet to understand the case without opening code.

Wire test IDs via annotation so the reporter back-fills results into the table automatically:

```ts
test('login succeeds with valid credentials @smoke @auth',
  { annotation: [{ type: 'case', description: 'TC-LOGIN-001' }] },
  async ({ loginPage }) => { /* ... */ });
```

Tables in `testcases/*.md` must have exactly these three column names: `TC-ID`, `Result`, `Notes`.
Missing columns → the reporter skips the file entirely. See `testcases/README.md`.

### Multiple people working on one feature

Split by **business flow**, each person gets their own Page Object files and their own ID
range, declared at the top of `testcases/<system>/<feature>.md`.

When asking AI to write tests while multiple people work on the same feature, **state the
scope explicitly**: which files can be touched, which ID range to use — otherwise AI may
edit another person's files.

**Reducing conflicts**: each person works on their own branch (no direct commits to `master`),
commit small and often rather than dumping a whole session into one large commit. Rebase only
when the working tree is clean (before opening an MR) — `git pull --rebase` is blocked when
you have uncommitted changes.

**Google Sheet has no merge like git**: two people pushing/pulling the same feature's Sheet
tab at the same time → the later push overwrites the earlier one in the overlapping cells.
Coordinate upfront who works on which section in a given session.

## Standard workflow when using the agent to generate tests (IMPORTANT)

The Playwright `playwright-test-generator` agent generates inline locator calls
(`page.click(...)`) directly in the spec. That is **not** this repo's convention. Because the
official agent files must not be edited (they get overwritten on every Playwright upgrade),
we handle this with a 3-step workflow:

1. **Generate the draft**: run planner → generator normally. The result is a raw spec with
   locators that have been verified against the real app (that's the agent's biggest value —
   real locators, not invented ones).
2. **Convert to POM**: use the `refactor-to-pom` skill to move the locators from the raw spec
   into the corresponding Page Object; the spec is left calling only methods + assertions.
3. **Check**: `npm run lint` must pass (the `no-raw-locators` rule catches any leftover raw
   locators in the spec) and `npx tsc --noEmit` must pass.

A raw spec that hasn't gone through step 2 **must not be merged**.

## Official agents — mandatory warnings

| Agent | What it's for | Warning |
|---|---|---|
| `playwright-test-planner` | Explore the app, generate a Markdown test plan in `specs/` | Can only see screens it can reach; knows nothing about business rules → **a human must review the plan before generating code** |
| `playwright-test-generator` | Generate specs from a plan, with verified real locators | Generates code that doesn't follow POM → must go through step 2 above |
| `playwright-test-healer` | Suggest fixes for broken tests when the UI changes | **HIGH RISK**: it may weaken assertions to match a broken app, or use `test.fixme()` to skip — hiding real bugs. **Never let the healer auto-merge. A human must read every line of its diff.** |

## Never fake a passing test

Fixing a failing test by relaxing the condition **hides a real bug** — more dangerous than
leaving the test red. The `healer` agent does this often.

| Forbidden | Why |
|---|---|
| Changing the expected value to match what the broken app shows | If the app is wrong, this hides the bug |
| `test.fixme()` / `test.skip()` to get through CI | A disabled test = a bug nobody sees |
| Replacing a specific assertion with `toBeTruthy()` | Almost always true, checks nothing |
| `expect.soft()` | The step still shows green even when it fails |
| `{ force: true }` to click a covered button | Also hides a UI bug |

`npm run check:tests` blocks these automatically. When genuinely needed, add a reason on
the line above:

```ts
// Reason: account-locking feature not yet implemented, waiting for ticket TICKET-123
test.fixme('block login when account is locked @auth', ...);
```
