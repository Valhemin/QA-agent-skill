---
name: release-readiness
description: Use when QA asks for an overall release-readiness summary — "any bugs before release", "ready to release", "summarize test cases" — not for running new tests, only summarizing existing testcases/ and the last run.
---

# Pre-release summary

Unlike `run-tests` (runs new tests, reports one run's result): this skill **doesn't run
tests**, it only summarizes existing status so QA can decide whether to release.

**Input:** nothing extra needed (reads all `testcases/**/*.md` + `reports/ai/summary.json` if present).
**Output:** a text summary report (no file created).

## Steps

0. Ask QA: "Want to sync the latest results from Sheet before summarizing?"
   - Yes → `npm run sheet:pull -- --feature=<feature> --sprint=<number>` for each needed feature.
   - No / skip → state clearly in the report: *"Results are based on the current md files,
     not synced from Sheet"* so QA knows there may be a gap.

1. List every `testcases/<system>/<feature>.md` file. For each, count the **Result** column:
   `✅ Pass` / `❌ Fail` / `⏭️ Skip` / `Not run` / `Manual`.

2. If `reports/ai/summary.json` exists, also pull `features` + `failed_tests` from the latest
   run to know which failures already have a bug ticket and which haven't been reported yet.

3. Summarize by system: total cases, % pass, list of **features with Fail or not-run cases**
   (name the specific case, don't just count).

4. **Give a recommendation, don't decide for QA**:
   - Fail/not-run cases on the main flow (`@smoke`) → state it clearly, suggest "not ready
     to release" with the reason.
   - Only secondary cases or "Manual" cases fail (e.g. depends on a third party) → state it
     clearly for QA to weigh, don't default to blocking the release.
   - "Not run" cases sitting for a long time with no clear reason → list separately, ask QA
     whether to drop or keep them.

5. No `reports/ai/summary.json` (no recent automation run) → state clearly "no recent run
   data", don't guess the result.

## Never do this

- Never conclude "ready to release" or "not ready" yourself — that's QA/lead's decision,
  this skill only provides data and a recommendation.
- Never track whether an external bug ticket (GitLab issue) is closed or not — this repo
  only tracks test results, not ticket status. QA checks ticket status on GitLab themselves.
