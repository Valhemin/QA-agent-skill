---
name: plan-testcases
description: Use when QA briefs requirements for a feature (fields, validation rules, buttons, expected behaviours) and wants a reviewable test case table before any code is written. Surveys the real UI, cross-checks against the brief, produces testcases/<he-thong>/<feature>.md for QA to review.
---

# Building a test case table from a requirement

This is **steps 1–3** of the team's process: QA briefs the requirement → AI surveys the real
UI → builds the test case table → QA reviews and edits → AI updates. **No code is generated
at this stage.**

**Input:** QA's verbal brief (fields, validation, buttons). **Output:**
`requirements/<system>/<feature>.md` (step 1) + `testcases/<system>/<feature>.md` (step 3).

Read `.claude/rules/edge-cases.md` — 7 edge-case groups + the 6-step process.

## Step 1 — Receive the brief, determine feature type, save docs

QA describes: what fields, character limits, text/number type, checkboxes, when buttons
appear, what happens on click.

**Ask a batched question right at the start of this step** (before doing anything else) —
combine everything into one message, don't ask piecemeal:
- What's the current sprint? (needed to name the Sheet tab, ask once per session)
- Any missing requirement-clarification questions (see list below)

Sprint already known from an earlier message in this session → **don't ask again**.

**First: determine single-screen vs multi-screen.**

| Signal | Type |
|---|---|
| One screen, one URL (e.g. login modal, add-address form) | **Single screen** — use the regular template |
| Multiple screens in sequence: step 1 → step 2 → ... → result (e.g. checkout flow, multi-step registration, payment wizard) | **Multi-screen** — see below |

**If multi-screen**: replace the "Screen"/"Figma design" lines in the template with the
"Screens in flow" table (see `.claude/templates/requirement.example.md`). Ask QA:

> "How many sequential screens does this feature have? What's each screen's URL, and does
> each have its own Figma link?"

Write "Direct entry" as `TBD` for now — fill it in after the UI survey in step 2.

Save to `requirements/<system>/<feature>.md` following the template in `requirements/README.md`.
**Why save it**: later, when a test fails, we need to know "what was actually expected" —
without this doc, nobody can judge whether the test is wrong or the product is wrong.

Brief is missing info → **ask right here**, don't wait until writing code:
- Field required or optional?
- Length limit? Any forbidden characters?
- Exact wording of the error message shown?
- Where does it go after success?
- **Does this feature need anything from another feature to work?** (e.g. needs login, needs
  a product, needs an existing order...) Write it into "Dependencies" in requirements.
  Don't know how to set it up yet → write TBD and ask during the UI survey
  (step 2 may reveal which API gets called during manual interaction).
- **Does the feature have time-based rules?** (expires, starts/ends at a time, locks after
  X minutes) → ask whether it's evaluated client-side (browser JS) or server-side, write it
  into "Business rules". Use `page.clock` if client-side, a mid-test API call if
  server-side (`writing-tests.md`).
- **Does the feature involve file upload?** (image, CSV...) → ask about allowed
  format/size, use sample files in `data/files/` or create new ones.
- **If this feature breaks, what's the worst outcome?** (e.g. "customer can't buy", "price
  calculated wrong", "displays wrong but order still goes through") → write into the Risk
  column of the test case table (High/Medium/Low). Used to decide `@smoke` vs `@regression`:
  any case that immediately blocks the user from using the product → High risk → `@smoke`.
  Otherwise → `@regression`.

No answer yet → write `TBD` in the doc, **never fill in a guessed value**.

## Step 2 — Survey the real UI

**Read `knowledge/<system>.md` + `knowledge/<system>/<feature>.md` (if it exists) first** —
if you already have the info, use it, don't re-survey from scratch.

Open the real screen on the system's URL (`.claude/rules/running-tests.md`) via Playwright
MCP. **This is a survey, not a test run** — don't record Pass/Fail, don't count it toward
`reports/ai/summary.json`. The official run (with results) only happens once, at step 5
(`write-test` → `run-tests`), after real code is generated.

Record **exactly what you see**:

- List of input fields, types, required or not (asterisk), placeholder
- Buttons/links, enabled/disabled state
- Actual error messages (exact wording, don't paraphrase)
- Navigation after a successful action

While surveying, **record locators into the "Locators recorded during survey" section in
`requirements/<system>/<feature>.md`** (see the template's example at the bottom):
- Locator priority: `getByRole` → `getByLabel` → `getByTestId` → `getByText` (full details
  in `.claude/rules/writing-tests.md` → "Code rules").
- Note the context if an element only appears under certain conditions.
- Clearly mark "Not guaranteed to still be correct if UI changes" — `write-test` will verify
  again when writing code; don't rely on this to skip the verify step, only to skip opening
  the browser a second time.

**Compare the brief against reality.** Discrepancy found → add it to "Needs clarification",
**don't decide which side is right**:

> Brief says the minimum password length is 8 characters, but the real screen accepts 6.
> Needs confirmation: is the requirement wrong, or is the product wrong?

This is often **a bug found before writing a single line of test code** — surface it clearly to QA.

**Immediately write to `knowledge/<system>/<feature>.md`** what you just observed (real URL,
sample locators, hidden flows, exact error messages) with a date, marked "verified" — so
`write-test` doesn't have to survey again from scratch.

**Multi-screen feature**: for each screen (except the first), fill in "Direct entry" in the
"Screens in flow" table — observe the network tab/API while interacting with that screen
manually, or ask QA/dev if there's a way to set state and `goto` directly (e.g. an API that
creates a cart, an existing `storageState`). Can't find a way → write `TBD`, **don't
block steps 1–3**, keep building the case table; only the edge cases for that screen will
have to navigate from the start when writing code later (slower but not wrong).

## Step 3 — Build the test case table

Write to `testcases/<system>/<feature>.md` following the exact template in
`testcases/README.md` (must include columns **`TC-ID`**, **`Result`**, **`Notes`** — exact
names, or the reporter can't fill them in).

ID rule: `TC-<FEATURE>-<3-digit number>`, e.g. `TC-LOGIN-001`.
Once assigned, an ID **never changes and is never reused**, even when a case is deleted —
prevents history confusion.

**Multi-screen feature**: organize the table by screen — each screen gets its own section
(`### Screen 1: <name>`), with cases belonging to that screen nested inside. The first screen's
section includes the end-to-end happy path (TC-001). Why: easier for QA to review ("I'm
reviewing the payment screen"), and `write-test` knows which spec to write first. Don't
flatten the table when a feature has multiple screens.

Cover all **7 edge-case groups** from `.claude/rules/edge-cases.md`. For each group:
- Brief already mentioned it → create the case as specified.
- Brief didn't mention it but the group applies → **still create the case, mark it
  `[AI suggestion]`** so QA knows this is an AI addition, not from the brief.
- Group doesn't apply (e.g. feature doesn't call an API → skip group 6) → state the reason.

Each case marks the `Automation` column: `✅` (will be automated) or `❌`
(manual-only — e.g. needs a real email check, a real device).

**Mandatory self-check before saving the file**: re-read every row, the `Steps` column must
never be blank for any case (including `❌`/`⏸️`) — see `.claude/rules/writing-tests.md` →
"Column Steps". Fill in any gaps before reporting to QA; don't let QA discover it later.

At the end of the table, summarize:
```
Total: <n> cases · Automated <a> · Manual <b>
From brief: <x> · AI-suggested: <y>
```

**Once the table is built → automatically push to Sheet right away, don't ask, don't wait
for QA to say "push it":**

```bash
npm run sheet:push -- --feature=<feature> --sprint=<sprint number asked in step 1>
```

Report the result with the name of the Sheet tab created/updated — **QA reviews on Sheet**,
not the md file (reason: QA is used to working in Sheet, dropdowns/formatting are easier to
read than raw markdown).

## Step 4 — QA reviews on Sheet, AI updates the md

QA opens the Sheet tab, reads the table, edits/adds/removes directly there. When done, they
tell AI (e.g. "I've approved it", "go ahead") — AI **never assumes QA is done**, waits for
explicit confirmation.

Confirmation received → automatically pull it back into the md, don't ask again:

```bash
npm run sheet:pull -- --feature=<feature> --sprint=<sprint number used when pushing>
```

Diff the old/new md to see what QA changed. Cases QA added manually on Sheet → already added
to the md (the script handles this), nothing extra to do. **Don't make unrelated edits** to
other cases beyond what QA just changed on Sheet.

Only move on to `/write-test` to generate code after the table has been pulled back.

## Never do this

- **Never generate test code in this skill.** Stop at the test case table. Code generation
  is `/write-test`.
- **Never invent an error message.** Haven't surveyed it yet → write `TBD`, don't make
  up something that sounds plausible.
- **Never decide whether the brief or the product is wrong** when they differ — add it to
  "Needs clarification".
- Don't create duplicate-intent cases just to make the table longer. Each case checks one
  distinct thing.
