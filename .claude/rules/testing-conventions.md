# Testing conventions — qa-automation

Source of truth for AI (skills/agents) and humans writing or reading tests in this repo.
This file covers rules that apply to **every** task. Details for each specific task live in
separate files — read only the file you need, no need to load them all each time.

> This file is owned by the team. `npx playwright init-agents` DOES NOT touch it
> (it only overwrites `.claude/agents/playwright-test-*.md` and `.mcp.json`). Therefore
> all team-specific rules must live here. DO NOT edit the 3 official agent files directly.

## Table of contents — read only what you need

| Task | File |
|---|---|
| Write/fix test code, naming, tags, generator/healer agents | `.claude/rules/writing-tests.md` |
| Build a test case table, review edge cases | `.claude/rules/edge-cases.md` |
| Run tests, accounts, leftover data, output | `.claude/rules/running-tests.md` |
| Compare UI against Figma | `.claude/rules/figma.md` |

---

## 0. Rule #1 for AI: never guess

Applies to every skill/agent in this repo. Takes priority over everything below.

- **Never guess.** Not sure → stop and ask the user. Don't infer and keep going.
- **Don't answer to fill space.** No data → say "no data", don't fabricate results or numbers.
- **Something unclear or missing → SET IT ASIDE, add it to "Needs clarification" at the end
  of the report**, then keep working on the rest. Don't stop everything; don't guess the
  unclear part.
- Every report ends with:
  ```
  ## Needs clarification
  - <skipped item> — <what's missing / who needs to decide>
  ```
  If nothing, write "None".
- Only state what you've verified with a real command/file. If you haven't run it, say so.

---

## 0b. Memory — read first, write after (MANDATORY)

`knowledge/<system>.md` stores what we know about each system in your project. Without it,
every session re-discovers the same things by trial and error.

| System | File | Domain |
|---|---|---|
| Web shop | `knowledge/web.md` (common) + `knowledge/web/<feature>.md` (feature-specific) | `app.example.com` |
| Admin | `knowledge/admin.md` | `admin.example.com` |
| Affiliate | `knowledge/affiliate.md` | `affiliate.example.com` |

### Before working on a system

**Read in order**: the common file first (`knowledge/<system>.md`) → then the feature-specific
file (`knowledge/<system>/<feature>.md`) if it exists. If you have the info, use it — don't
re-survey. Only survey when the file doesn't exist or the info in it is wrong.

### Two-tier structure — split from the start

`knowledge/<system>.md` — **system-wide** info: navigation patterns, noisy requests,
performance, anything not specific to one feature.

`knowledge/<system>/<feature>.md` — **feature-specific** info: locators, flows, API calls,
validation behaviour. Create this file as soon as you survey a new feature; don't dump
everything into the common file.

Why split early: multiple QAs working on the same system → each adds a new feature by
creating only their own file, no one touches anyone else's → no conflicts.

### Identifying which system you're working on

Most reliable source: **file path** `testcases/<system>/<feature>.md` — reporters scan the
filesystem; they don't read static config (the `features` field has been removed from
`data/he-thong.json`).

When the user says it explicitly ("test web", "check the admin") follow their words, even
if the URL they paste looks different — QA often pastes a link to an admin screen hosted on
a shared domain.

### Detecting stale memory

When tests fail in the `locator` group (element not found) → the UI has likely changed →
the reporter writes a warning to `reports/ai/summary.json` → `knowledge_warnings`, pointing
to exactly the right system file.

**Only warn on error**, not on a schedule — avoid noise.

Seeing a warning → **re-read the file, compare against the real screen, fix immediately**.
Don't leave stale info: the next session will trust it, repeat the same mistake, and waste
exactly the time memory was supposed to save.

### After learning something new — write it down immediately

Must record when:
- You just **failed then succeeded** (clicked the wrong hidden element, URL returned 404,
  needed an intermediate step)
- You found a multi-step flow that's impossible to guess without seeing it
- QA stated a business rule during work
- Old info in the file turned out to be wrong → **fix it now**, note "changed YYYY-MM-DD"

Each entry gets a date and status: "verified" or "QA-described, unverified".
**Only record what you actually saw.** Don't record guesses — one wrong entry
propagates errors indefinitely.

### System relationships

Affiliate **has no login form** — you must log in on web first; the shared-domain cookie
carries over automatically. This means affiliate tests must reuse the web login session
(`storageState`); don't write a "login to affiliate" test.

### Distinguishing the directories

| Directory | Contains | Location pattern |
|---|---|---|
| `knowledge/` | How the system **actually works** — observed facts | `<system>.md` |
| `requirements/` | How the system **should work** — QA brief | `<system>/<feature>.md` |
| `testcases/` | What to check + run results | `<system>/<feature>.md` |

`tests/` and `pages/` are organized by feature only (`tests/checkout/`), not by system —
the `@checkout` tag is enough to distinguish.

A gap between `knowledge/` and `requirements/` is a **bug**.

---

## Docs must match code

Skills/rules that mention field names in `summary.json`, npm commands, or file paths — if
the code changes but the docs don't, AI reads the wrong thing, reports break, and nobody
notices.

`npm run check:docs` checks this automatically. Rename a reporter field or change a script
in `package.json` → run this before committing.

## After upgrading Playwright

`npx playwright init-agents --loop=claude` overwrites `.claude/agents/playwright-test-*.md`
and `.mcp.json`. After running it, verify that `.mcp.json` still has the `Figma` entry —
if it's gone, add it back. (`tests/seed.example.spec.ts` and all rule files are NOT
overwritten — verified.)
