# qa-agent-kit

Portable AI QA agent skill kit for Playwright + TypeScript repos.

Copy the contents of this folder into a new repo and get a working AI QA agent that:
- Writes and runs Playwright tests
- Plans and reviews test cases
- Compares built UI against Figma designs
- Writes bug tickets from test failures
- Summarises release readiness

Compatible with **Claude Code** and **Cursor** (both read `.claude/` automatically).

---

## What's included

```
.claude/
├── commands/qa.md           — slash command (/qa) that routes every QA request
├── rules/
│   ├── testing-conventions.md  — core rules: no-guess, memory, table of contents
│   ├── writing-tests.md        — code rules, POM, naming, tags, agents
│   ├── edge-cases.md           — 7 mandatory edge-case groups per feature
│   ├── running-tests.md        — systems, accounts, leftover data, output format
│   └── figma.md                — Figma MCP tool order, breakpoints, discrepancy rules
└── skills/
    ├── qa/                  — single entry point, routes to the right sub-skill
    ├── cleanup-testdata/    — delete leftover test data (with confirmation)
    ├── figma-ui-check/      — accept/reject a built screen against Figma
    ├── plan-testcases/      — survey UI, build test case table, push to Sheet
    ├── refactor-to-pom/     — convert raw agent-generated specs to Page Object Model
    ├── release-readiness/   — summarise test results across all features
    ├── report-bug/          — write a ready-to-paste bug ticket
    ├── review-edge-cases/   — check a tester's case list against the 7 groups
    ├── run-tests/           — run Playwright, read summary, push results to Sheet
    └── write-test/          — write Page Object + spec, verified by lint + typecheck
data-samples/
    ├── feature-map.example.md      — feature → path/tag mapping table
    ├── he-thong.example.json       — system URLs (web/admin/affiliate)
    └── test-users.example.json     — test account structure by system and role
CLAUDE.md                    — template entry point for the new repo
```

---

## Prerequisites

- **Node.js** 18+
- **Playwright** with TypeScript: `npm install @playwright/test typescript`
- **Claude Code** CLI (`claude`) or **Cursor** with Claude
- Optional: Figma MCP server (for UI acceptance checks) — see [Figma MCP](https://mcp.figma.com)
- Optional: Google Sheets integration (for pushing/pulling test case tables)

---

## Installation

### Step 1 — Copy files into your repo

```bash
cp -r qa-agent-kit/.claude  your-repo/
cp    qa-agent-kit/CLAUDE.md your-repo/CLAUDE.md
```

The `data-samples/` files are not copied directly — see Step 3.

### Step 2 — Create the required folder structure

The skills expect this layout in your repo (create empty directories or let them appear as
you add features):

```
requirements/         # QA briefs: requirements/<system>/<feature>.md
testcases/            # Test case tables: testcases/<system>/<feature>.md
knowledge/            # Observed app behaviour: knowledge/<system>.md
reports/
  ai/                 # summary.json written by reporters
  qa/                 # HTML reports, diagnostics written by reporters
data/
  feature-map.md      # copy from data-samples/feature-map.example.md, edit
  he-thong.json       # copy from data-samples/he-thong.example.json, edit
  test-users.json     # copy from data-samples/test-users.example.json, fill in (gitignore it)
tests/                # specs — one subfolder per feature
pages/                # Page Objects — one subfolder per feature
fixtures/             # Playwright test.extend() fixtures
utils/                # plain helper functions
```

### Step 3 — Fill in placeholders

Copy the example data files and customise them:

```bash
cp data-samples/feature-map.example.md  data/feature-map.md
cp data-samples/he-thong.example.json   data/he-thong.json
cp data-samples/test-users.example.json data/test-users.json
```

Then edit:

| File | What to fill in |
|---|---|
| `CLAUDE.md` | Replace `<PROJECT_NAME>` with your repo name |
| `data/he-thong.json` | Replace URLs with your real system domains |
| `data/feature-map.md` | Replace example features with your actual features |
| `data/test-users.json` | Add real test accounts (this file is gitignored) |

**Add `data/test-users.json` to `.gitignore`** — it contains credentials.

### Step 4 — Rename or drop systems you don't have

The kit ships with three systems: `web`, `admin`, `affiliate`. If your project has different
systems (e.g. only `web` and `admin`, or named differently):

1. Edit `data/he-thong.json` — rename or remove system keys.
2. Update `data/feature-map.md` — update paths under the right system.
3. Search `.claude/rules/running-tests.md` and `.claude/skills/qa/chi-tiet.md` for
   "affiliate" and adjust the cross-system login note to match your actual setup.
4. Add a note at the top of `knowledge/<system>.md` for each of your real systems.

### Step 5 — Add npm scripts to your package.json

The skills reference several `npm run` scripts. Add these to your `package.json` (adjust
commands to match your project):

```json
{
  "scripts": {
    "test:smoke":      "playwright test --grep @smoke",
    "test:regression": "playwright test --grep @regression",
    "test:failed":     "playwright test --last-failed",
    "test:ui":         "playwright test --ui",
    "test:debug":      "playwright test --debug",
    "test:responsive": "playwright test tests/responsive/",
    "lint":            "eslint .",
    "check:tests":     "node scripts/check-no-weakened-tests.mjs",
    "check:docs":      "node scripts/check-docs-match-code.mjs",
    "sheet:push":      "node --experimental-strip-types scripts/sheets-push.ts",
    "sheet:pull":      "node --experimental-strip-types scripts/sheets-pull.ts",
    "sheet:push-result": "node --experimental-strip-types scripts/sheets-push-result.ts"
  }
}
```

`check:tests` and `check:docs` are optional guard scripts — the source repo contains
reference implementations. `sheet:*` are only needed if you use Google Sheets sync.

### Step 6 — Wire up reporters (optional but recommended)

The `run-tests` and `release-readiness` skills read from `reports/ai/summary.json`. That
file is written by a custom Playwright reporter. See `reporters/summary-reporter.ts` in
the source repo for the reference implementation, or write a reporter that produces:

```jsonc
{
  "total": 12,
  "passed": 10,
  "failed": 2,
  "duration_ms": 34200,
  "base_url": "https://app.example.com",
  "features": [...],
  "failed_tests": [...],
  "knowledge_warnings": [],
  "leftover_data": [],
  "next_steps": [...]
}
```

### Step 7 — Run your first test

```bash
# Run all tests (uses chromium by default)
npx playwright test

# Run only smoke tests
npm run test:smoke

# Run a specific feature
npx playwright test --grep "@auth"
```

---

## What to customise

| Placeholder / default | Where | What to change |
|---|---|---|
| `<PROJECT_NAME>` | `CLAUDE.md` | Your repo name |
| `app.example.com` | `data/he-thong.json`, `knowledge/` | Your real domains |
| `admin.example.com` | same | Your admin domain |
| `affiliate.example.com` | same | Your affiliate/partner domain, or remove if none |
| `web / admin / affiliate` | system names throughout | Your actual system names |
| Feature rows in `data/feature-map.md` | `data/feature-map.md` | Your real features |
| Test accounts | `data/test-users.json` | Real credentials per system + role |
| Google Sheets `spreadsheetId` | `data/he-thong.json` | Your Sheet ID, or remove if not using Sheets |

---

## How the agent handles QA requests

The user speaks plain language. The `/qa` command (or calling the `qa` skill) routes the
request to the right sub-skill automatically:

| User says | Skill used |
|---|---|
| "run test X", "any failures?" | `run-tests` |
| "ready to release?" | `release-readiness` |
| "write a test for X" | `write-test` |
| Full requirements brief | `plan-testcases` |
| Pastes a list of cases, "what's missing?" | `review-edge-cases` |
| Figma link, "compare with design" | `figma-ui-check` |
| "write up this bug for dev" | `report-bug` |
| Spec has raw locators | `refactor-to-pom` |
| "clean up leftover test data" | `cleanup-testdata` |

The agent never makes the user pick a skill or type a command.

---

## Customising the system names

The skills use "web / admin / affiliate" as example names throughout. To rename them:

1. Do a global find-replace in `.claude/` for the system names you want to rename.
2. Update `data/he-thong.json` keys to match.
3. Update `knowledge/` filenames and any `testcases/` / `requirements/` folder names.

The skills reference system names via `data/he-thong.json` at runtime, so renaming there
plus the knowledge files is enough for the agent to behave correctly.
