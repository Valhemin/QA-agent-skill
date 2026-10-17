# Details — read only when you hit the relevant task

The everyday content is in `SKILL.md`. This file is split out to avoid loading it every session.

## Test accounts

Declared in `data/test-users.json` as **system → role**. Fetch with `getUser`:

```ts
import { getUser } from '../../utils/users';

const u = getUser('web', 'mua-hang');
await loginPage.login(u.email, u.password);
useAccount(u);   // REQUIRED: so the repro steps tell the dev exactly which account to log in with
```

- **QA sends a list of accounts** → add them to the file, naming each role in kebab-case
  that describes its purpose (e.g. `buyer`, `locked-account`, `shop-owner`), with a `note` field.
  State clearly if overwriting an existing role.
- No real account yet → `getUser` throws with instructions; the report lists it under
  `missing_accounts`. When that happens: state which account is missing and what it's for,
  ask QA to provide it. **Never invent one.**
- File is gitignored (never commit real credentials). Template: `data/test-users.example.json`.

## Performance and accessibility

```ts
import { doHieuNang, quetTiepCan } from '../../utils/kiem-tra-chat-luong';

const hn = await doHieuNang(page);     // LCP, CLS, load time
const loi = await quetTiepCan(page);   // serious WCAG 2 AA violations
```

Core Web Vitals thresholds (web.dev): LCP < 2500ms, CLS < 0.1.

Use when QA asks "is the page slow", "check accessibility", or when accepting a new screen.
**Don't** add to every test — it slows the suite with little extra value.

Measurements depend on network and machine. One reading above the threshold → **say clearly
it's a single measurement**, suggest re-measuring a few times before calling it a bug.

## Leftover data on real environments

Tests run against a deployed domain, so creating orders/accounts leaves real data behind.

```ts
trackData({ loai: 'don-hang', id: orderId, xoaBang: 'Admin > Orders > Cancel' });
```

Tests that can self-clean do so in `afterEach`, marked `daDon: true`. `trackData` is only
for items that can't be self-cleaned.

After a run, `reports/ai/summary.json` → `leftover_data`. If non-empty: list it in the report
→ **ask the user whether to delete**. Confirmed → use the `cleanup-testdata` skill.
**Never delete on your own** — they may still need it for manual inspection.

## Cross-system login

Affiliate **has no login form** — log in on web first; the shared-domain cookie carries over.
Affiliate tests must reuse the web login session (`storageState`), **not** write a
"login to affiliate" test. Details: `knowledge/affiliate.md`.

Admin **does have** its own `/login` page, separate from web. Don't use `pages/auth/LoginPage.ts`
from the web system for admin.

## Multiple people working on one feature

Only edit the files you've been assigned; use only the ID range allocated to you.
Scope not stated → ask before touching anything.
