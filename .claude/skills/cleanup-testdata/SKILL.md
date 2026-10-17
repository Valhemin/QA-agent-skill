---
name: cleanup-testdata
description: Use when QA asks to clean up leftover test data created on a real environment (orders, accounts, addresses created by automated tests that weren't auto-cleaned) — reads reports/qa/test-data.json, confirms with the user, then deletes.
---

# Cleaning up leftover test data on real environments

Use when QA says "clean up the test data", "any leftover test data", or after
`reports/ai/summary.json` → `leftover_data` is non-empty and QA wants to handle it now.

Read `.claude/rules/running-tests.md` — "Leftover data on real environments" section.

## Rule #1: never delete on your own

AI **never** deletes without QA confirming each group. There's no "delete all, report
after" mode — the data may be something QA still needs for manual testing.

## Steps

1. Read `reports/qa/test-data.json` (detailed file, `items` field, filter `daDon !== true`).
   File doesn't exist → fall back to `reports/ai/summary.json → leftover_data` (a summary
   without `xoaBang` — QA will have to delete manually). Both empty/missing → report
   "No leftover data recorded", stop.

2. Group by `loai` + `xoaBang`, show QA a table:

   ```
   | Type | ID | How to delete | Count |
   |---|---|---|---|
   | don-hang | DH-1023, DH-1024 | Admin > Orders > Cancel | 2 |
   | tai-khoan | test-abc@mail.com | unknown | 1 |
   ```

   Rows with `xoaBang: 'unknown'` or empty → set apart, tell QA they must delete manually
   since there's no instruction — don't guess a deletion method.

3. **Ask QA to confirm** — per group (e.g. "delete both orders above?"), not a single blanket
   question if there are multiple different types that need manual vs AI-via-Playwright-MCP handling.

4. For groups QA agreed to and with a clear `xoaBang` describing the admin action (e.g.
   "Admin > Orders > Cancel"): use Playwright MCP to open the corresponding admin domain
   (look it up in `data/he-thong.json`), perform the described action, confirm each item is
   deleted before moving to the next.
   - Can't find the screen/button as described → stop on that item, add to "Needs
     clarification", **don't guess another way to delete it**.
   - After acting, screenshot or re-read the list to confirm the item is actually gone —
     don't just trust that clicking returned no error.

5. After deleting (or QA deleted manually), update `daDon: true` for the processed items in
   `reports/qa/test-data.json`, overwriting the whole file. Keep other fields unchanged.

6. Final report: how many deleted, how many remain (and why — QA declined, or missing
   `xoaBang`), with a "Needs clarification" section for any action that couldn't be completed.

## Never do this

- Never delete without asking, even when `xoaBang` describes the method clearly.
- Never infer a deletion method when `xoaBang` says "unknown" — leave that to QA or ask dev.
- Never delete real data belonging to another user by mistake — only act on the exact IDs
  recorded in `test-data.json`, never delete by range or by "looks similar".
- Never run this step against a production domain without a separate confirmation (see
  `.claude/rules/running-tests.md` — production caution section).
