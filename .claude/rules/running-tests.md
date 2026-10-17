# Running tests — systems, accounts, leftover data, output

Read when: running tests (`run-tests`), writing a bug report (`report-bug`).

## Systems and URLs

Three systems, URLs come from `data/he-thong.json` (source of truth — not hardcoded here to
avoid drift):

| System | Specific behaviour |
|---|---|
| web | Root — log in here first, then into affiliate |
| admin | Has its own `/login` page, different from web — use `--project=admin` |
| affiliate | No login form — inherits the session from web via a shared-domain cookie |

Running tests:

```bash
npx playwright test --grep "@auth"                  # web (default)
npx playwright test --project=admin --grep "@admin" # admin
npx playwright test --grep "@auth|@duyet-don"       # multiple systems in one session
```

To change URLs, edit `data/he-thong.json`. To override temporarily (a different
environment) without editing any file: `BASE_URL=https://staging.example.com npx playwright test`.

To force a specific worker count (without `.env`), add `--workers=N`:
`npm run test:smoke -- --workers=1`. Normally Playwright auto-picks based on CPU count.

**Adding a new feature**: no need to edit `data/he-thong.json` — just create
`testcases/<system>/<feature>.md` in the right place; scripts/reporters scan the filesystem
to figure out which system a feature belongs to. This lets a failed-test report point to the
right `knowledge/<system>.md` — even when multiple systems are tested in one session.

**Caution with production**: ask for confirmation before running tests against the
production domain, since tests can create real data.

## Test accounts by role

Each feature usually needs a different account type. All declared in
`data/test-users.json`, organized as **system → role**:

```json
{ "web": { "mua-hang": { "email": "...", "password": "...", "ghichu": "..." } } }
```

- Fetch in a test: `getUser('web', 'mua-hang')` (from `utils/users.ts`).
  No real value yet → it throws with instructions, instead of a confusing test failure.
- Each system should have one `mac-dinh` (default) account shared across tests. A new feature
  doesn't need its own account right away — `getUser` falls back to `mac-dinh` automatically.
  Only add a dedicated account when the feature needs behaviour different from the shared one
  (e.g. a locked account).
- **Must call `useAccount(u.nhan)` after logging in** — otherwise the bug repro steps can't
  tell the dev which account to log in with.
- QA sends a list of accounts → AI adds them to the file, naming each role in kebab-case
  describing its purpose, with a `ghichu` (note). State clearly if overwriting an existing role.
- Accounts not yet filled in show up in `reports/ai/summary.json` → `missing_accounts`.
- **Never commit production passwords.**

## Leftover data on real environments

Tests run against a deployed domain, so every order / account / address created leaves
**real data** behind. Running repeatedly accumulates clutter, which can cause later tests
to fail.

- Tests that can self-clean → clean up in `afterEach`, mark `daDon: true`.
- Can't self-clean → record it with the `trackData({ loai, id, xoaBang })` fixture.
- After a run, `reports/ai/summary.json` → `leftover_data` lists what's left over.

**AI must never delete data on its own.** List it in the report, **ask the user**, only
delete after they agree and confirm. They might still need that data for manual checking.

## Standard output

After every run, `reports/ai/summary.json` is the **only** source AI reads to report to the
tester (don't read the raw HTML report, don't infer from terminal logs). Format: see
`reporters/summary-reporter.ts`.

`reports/` is split by reader: `reports/ai/` (summary, run history, triage data) — AI reads
this, QA doesn't need to open it; `reports/qa/` (HTML report, `.md` evidence on failure) — QA
opens this directly; `reports/artifacts/` is Playwright's internal folder (raw
trace/video/screenshot, `.last-run.json` used by `test:failed`) — nobody needs to open these
by hand, always read via `summary.json` or the HTML report.
