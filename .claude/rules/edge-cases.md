# Mandatory edge-case groups for every form/input feature

Read when: building a test case table (`plan-testcases`), reviewing cases a tester gave you
(`review-edge-cases`).

When writing tests or reviewing tester-provided edge cases, check against all the groups
below. Only add cases missing from the listed groups — don't invent cases outside the
feature's scope:

1. Happy path (valid input, main flow)
2. Empty input / missing required field
3. Wrong-format input (invalid email, negative number, special characters)
4. Boundary limits (min/max length, quantity 0, max quantity)
5. Special account/data states (locked, expired, deleted, duplicate)
6. System/network errors (API timeout, 500) — only if the feature calls an API that surfaces
   in UI state
7. Access control (user without permission, not logged in)

## 6-step process to automate a feature

The full 6-step process (brief → survey → plan → review → code → triage) is described in
your project's main README. This file only covers edge cases and responsive (the detail for
step 3 of that process).

## Responsive — ask at step 1, default to light-touch

When receiving a brief, ask this together with other questions:

> **"Does this need a thorough responsive check this time (running `test:responsive` across
> 5 breakpoints), or is it enough to confirm mobile/tablet/desktop render normally without
> breaking layout?"**

| Answer | Check level | Action |
|---|---|---|
| Needs thorough check (usually a new screen, freshly designed UI) | Full 5 breakpoints (sm 576 / md 768 / lg 992 / lg-2 1024 / xl 1200) | Create a spec in `tests/responsive/web/` or `tests/responsive/affiliate/`, run `npm run test:responsive` |
| Not needed / bug fix / minor addition | 3-tier (mobile/tablet/desktop) | **Nothing extra needed** — the `chromium`/`tablet`/`mobile-chrome` projects already run in parallel when `--project` isn't specified |

Admin has no responsive — desktop-only, don't ask.

When running 3-tier, if behaviour or display differs between mobile and desktop
→ **don't decide right/wrong yourself**, add it to "Needs clarification" with a screenshot
for QA to review.
