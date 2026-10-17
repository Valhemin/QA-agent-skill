---
name: run-tests
description: Use when user asks to run/re-run tests, run smoke, or run regression for a feature in this repo. Executes Playwright, reads reports/ai/summary.json, reports results concisely with next steps.
---

# Running tests

**Input:** feature name (+ domain if different from default). **Output:** a text result
report in the format below, read from `reports/ai/summary.json` (no new file created).

Read `.claude/rules/running-tests.md` — systems, accounts, leftover data, output.

## Read knowledge/ first

Read `knowledge/<system>.md` + `knowledge/<system>/<feature>.md` (if it exists) to learn
the URL, special locators, tricky flows — avoid re-surveying from scratch.

## Before running — one batched question, asked once

Ask immediately on receiving the request, bundled into one message:
- **Which round?** (needed to push results to the right column after the run — ask once,
  don't ask again)
- What's the current sprint? (if not already known in this session — needed for `push-result`)

Sprint and round already known in this session → **don't ask again**.

## Steps

1. Map the feature name → tag/path using `data/feature-map.md`.
   Feature **not in the table → stop, ask the user**. Don't guess the path.

2. Identify the domain (`.claude/rules/running-tests.md`). Tests run against a deployed
   domain, not a local app.
   - Default: URL from `data/he-thong.json` for the feature's system — no need to ask.
   - User specifies a different environment → `BASE_URL=<domain> npx playwright test ...`.
   - Domain is **production** → **ask for confirmation before running**, since tests can
     create real data.

3. **Before running `test:smoke`/`test:regression`** (large suites, takes time): ping the
   domain first with `curl -o /dev/null -s -w "%{http_code}" <domain>` or open one page via
   Playwright MCP. Domain dead/5xx → **stop immediately, report "environment is down, test
   not run"**, don't run the full suite and produce dozens of fake timeouts (wastes money,
   makes QA think there are many bugs). Running a single feature (`--grep`) → skip this
   step; it fails fast and that's enough signal.

4. Run exactly one command:
   | Request | Command |
   |---|---|
   | One feature | `npx playwright test --grep "@<feature>"` |
   | Smoke (PR) | `npm run test:smoke` |
   | Full regression | `npm run test:regression` |
   | Re-run last failures | `npm run test:failed` |

   **Headed/headless depends on the command type**:
   - `--grep "@<feature>"` (running one feature being worked on/debugged) → **browser open
     by default**, 1 worker, QA watches each step live.
   - `test:smoke` / `test:regression` / `test:failed` → **headless by default** (scripts
     set `HEADLESS=1`), Playwright picks worker count from CPU — many tests run in parallel,
     an open window can't be watched anyway and will slow the machine down.
   - CI: always headless automatically, nothing extra needed.

   Override when the user says so explicitly:

   | User says | Command | Note |
   |---|---|---|
   | "run headless", "run fast" (with single-feature command) | add `HEADLESS=1` before the command | No window, more workers |
   | "open browser for smoke" | `HEADLESS=0 npm run test:smoke` or drop the flag | 1 worker, slower |
   | "I want to watch / inspect myself" | `npm run test:ui` | **Holds the terminal until the user closes it** |
   | "debug step by step" | `npm run test:debug` | **Pauses immediately at step 1**, waits for user to press Resume in Playwright Inspector |

   **`test:ui` and `test:debug` are interactive modes — they don't exit on their own** —
   the window sitting there waiting is not a hang. Must warn the user before running either
   ("this command opens a window and waits for you, close the window or Ctrl+C when done").
   AI can't read the result since the user is in control — finish by saying "check the
   results in that window", don't fabricate results.

   **Save tokens**: DON'T read the full terminal output. Add `> /dev/null 2>&1` then read
   `reports/ai/summary.json` — that file has everything needed, much shorter than hundreds
   of log lines.

5. Read **only** `reports/ai/summary.json`. Don't read the raw HTML report, don't read
   terminal logs, don't open `trace.zip` (binary file, useless to AI and wastes tokens).
   File doesn't exist → report that clearly, **don't guess the result**.

6. Only when failures are in the `assertion` or `timeout` group: read the matching file in
   `reports/qa/diagnostics/` to see what the screen actually showed and which requests
   errored. **At most 2 files per run** (pick the first failing test from each group) — many
   tests usually share the same root cause, reading all of them is wasteful.

7. Check `knowledge_warnings` in summary.json — if there's a warning, re-read
   `knowledge/<system>.md` or `knowledge/<system>/<feature>.md`, compare against the real
   screen, **fix the knowledge file immediately** before reporting. Stale memory left in
   place will cause the same mistake next session.

8. Write the report using the template below.

## Report template (concise, just enough)

```
<passed>/<total> pass · <failed> fail · <duration>s · <domain>

[if failures — group tests with the same root cause, DON'T list each test separately]
▸ <cause group> (<count>)
  <short description of what happened, based on real evidence>
  File: <path:line>

Needs manual action:
  - <file> — <what to do>

Next steps:
  1. <highest priority>
  2. <next>

## Needs clarification
- <unclear item> — <what's missing>
```

Reporting rules:
- **Max 15 lines** for the result section. Longer and QA won't read it.
- Group failures with the same cause into one line ("3 tests fail because the Checkout button
  was renamed"), don't list 3 identical lines.
- Use `next_steps` and `files_need_attention` already in `summary.json` — don't rethink them.
- Use business language, don't paste stack traces. QA needs to know "what broke", not
  internal Playwright function names.
- Nothing unclear → write "None".

## Meaning of failure groups in summary.json

| Group | Meaning | How to handle |
|---|---|---|
| `environment` | Wrong domain / app down / network | **Not a product bug.** Fix the environment, re-run |
| `locator` | Element not found | UI changed → fix locator in `pages/` |
| `assertion` | Element found, value wrong | **Suspect a real bug** → read diagnostics then report |
| `timeout` | Waited too long | App slow / API hanging → check request errors in diagnostics |
| `flaky` | Passed on retry | Unstable test, review the wait strategy |

## After the run — automatically push results to Sheet

No further questions (round + sprint already asked at the start). After step 8 (summary.json
read), run immediately:

```bash
npm run sheet:push-result -- --feature=<feature> --round=<round asked at start> --sprint=<sprint asked at start>
```

After the push, include in the final report:
```
<passed>/<total> pass — Round <N> results pushed to Sheet.
QA can view at tab "[Sprint <N>] Automation test: <System> - <Feature name>".
[if failures] <n> cases Failed — Attachment + repro written to the corresponding columns.
```

**Don't wait for QA to say "push the results"** — pushing results is the default final step
of run-tests.

## Never do this

- **Never change code to force a pass** unless the user explicitly asks.
- Never hide a failing test in the report, even if you suspect it's flaky — state the
  suspicion clearly.
- Never conclude "product has a bug" when the group is `environment` — that's an
  infrastructure issue.
- Never conclude "test is wrong" when the group is `assertion` without reading diagnostics:
  the product may genuinely be broken. Not sure → add it to "Needs clarification" for QA to decide.
