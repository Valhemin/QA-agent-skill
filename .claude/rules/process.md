# Full Feature Workflow

Read when: you want the standard 6-step order, or when a brief changes mid-flight.

| Step | What | Who | Output |
|---|---|---|---|
| 1 | Describe requirements (fields, validation, buttons) | QA + AI | `requirements/<system>/<feature>.md` |
| 2 | Survey the real UI, **report every deviation** from step 1 | AI | "Needs clarification" section if deviations found |
| 3 | Create test case table, cover all 7 scenario groups | AI | `testcases/<system>/<feature>.md` |
| 4 | **QA reviews the table** — edit, add, remove | QA | Updated table |
| 5 | Write code, run it, fill in results in the table | AI | `tests/`, `pages/` |
| 6 | Fail → classify, provide repro steps, suggest routing to FE/BE | AI | Bug report |

**Step 2 often finds bugs before a single test is written.** Example: QA says minimum password is 8 characters, AI tries 6 and the app accepts it → report immediately: *"Is the requirement wrong or is the product wrong?"*

Two points requiring QA decision: **table review** (step 4) and **manual verification before filing a bug** (step 6).

**UI acceptance against Figma** is not a numbered step — run it **in parallel**, any time after step 2 (once there is a real UI to compare), typically when dev says "screen X is built." Use `skills/qa-figma/SKILL.md`; it does not block the 6 main steps.

## Requirement Changed Mid-Flight (not a new feature)

Unlike a new feature (restart from step 1), an existing requirement change means **do not rewrite from scratch**:

1. Edit only the changed part in `requirements/<system>/<feature>.md`; annotate the date inline (e.g. "Changed 2026-10-20: 8 chars → 6 chars").
2. Find related cases in `testcases/<system>/<feature>.md` by TC-ID or description matching the changed part — **do not scan the whole table if only 1–2 fields changed**.
3. For related cases: update the description/expected result, reset `Result` to `Not run` (old value no longer applies). Keep the TC-ID; do not issue a new ID for an in-place edit.
4. Open the matching `tests/<feature>/*.spec.ts`, update assertions/data to match the new requirement, run to green before reporting done — same as `skills/qa-write/SKILL.md`.
5. If many things changed (more than a few fields, the whole flow changed) → treat as a new feature and restart from step 1.

## Two People on the Same Feature

Split by **business flow** — each person owns separate Page Object files and a separate TC-ID range. Document the split at the top of `testcases/<system>/<feature>.md`.

When asking AI to write tests while multiple people are working on the feature, **be explicit about scope**: which files can be edited, which TC-ID range to use — otherwise AI may touch files another person is actively editing.

## When a system accumulates too many features in `knowledge/`

If `knowledge/<system>.md` exceeds ~150 lines → split by feature: `knowledge/<system>/<feature>.md`. Keep one root file that is only a table of contents pointing to each feature file. `tests/` and `pages/` are always split by feature (`tests/checkout/`), not by system — the `@checkout` tag is sufficient to differentiate.
