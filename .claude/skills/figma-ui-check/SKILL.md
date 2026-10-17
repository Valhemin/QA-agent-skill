---
name: figma-ui-check
description: Use when dev has finished building a screen and QA needs to verify the built UI matches the Figma design — wrong text, missing button, wrong layout, responsive issues. Compares Figma frame against the real deployed screen at 3 breakpoints.
---

# UI acceptance: dev-built screen vs Figma design

Use when **dev has finished building a screen** and QA needs to verify the real screen on
staging matches the Figma design: wrong text, missing button, wrong layout, broken responsive.

**Output:** acceptance report (template below) + screenshots at 3 sizes saved to
`reports/figma-check/<feature>-{desktop,tablet,mobile}.png`. No code is created or modified.

Read `.claude/rules/figma.md` — breakpoint rules, node-id parsing.

## Required inputs

**First**: check whether `requirements/<system>/<feature>.md` has a "Screens in flow"
table. If yes → this is a multi-screen feature; follow the "Multi-screen feature" section
below instead of the single-screen flow.

| Need | If missing |
|---|---|
| Figma link to **the exact frame** (with `?node-id=`) | Whole-file link → **ask which frame**, don't pick one yourself |
| Real screen path (e.g. `/checkout/payment`) | **Ask**, don't guess the URL |
| Domain (staging/production) | Default from `data/he-thong.json`; production → **ask for confirmation** |

## Steps

0. Read `knowledge/<system>/<feature>.md` if it exists — learn the real URL, whether login is
   required, or any breakpoint quirks recorded previously.

1. Read the Figma frame via MCP **in the order** defined in `.claude/rules/figma.md`:
   `get_metadata` first (light outline), then `get_screenshot` (visual comparison), only call
   `get_design_context` when you need precise numbers (spacing/colour/font) — don't call all
   tools "just to be safe".
   Can't read it (no access, wrong link) → **state the reason clearly, stop, ask for a new link**.
   Never estimate numbers from a screenshot.

2. Open the real screen with Playwright MCP at **desktop 1280px**, capture `ariaSnapshot`
   (text-based content tree) + screenshot.

3. **Compare content first, layout second** — this priority matters because a missing button
   is more severe than a 2px offset:

   a. **Missing/extra elements**: buttons, inputs, links, images in Figma but not on the real
      screen (or the reverse). Most severe category.

   b. **Wrong text**: compare each string. Watch for typos, wrong capitalisation, missing
      Vietnamese diacritics, placeholder text ("Lorem ipsum", "Text here") still showing.

   c. **Wrong state**: a button that should be disabled but is enabled, missing loading/empty/
      error states that Figma designed.

   d. **Layout/colour/font mismatches**: only note when the discrepancy is large enough to see.

4. Repeat steps 2–3 at **tablet 768px** and **mobile 375px**. At these sizes focus on:
   broken layout, text overflow, buttons covered or untappable, horizontal scroll required,
   distorted images.
   Figma has no design for that size → write "Figma has no design for this size", **don't
   infer** what the small screen should look like from the desktop frame.

5. **New screen (never accepted before):** run 2 extra quality checks without QA having to
   ask. How to call them: see `qa/chi-tiet.md` — "Performance and accessibility" section.

   Add these two sections to the report after the layout issues:

   ```
   ▸ Performance (LCP / CLS)
     LCP: <ms> (<Pass if <2500> / Warning if ≥2500>)
     CLS: <value> (<Pass if <0.1>)

   ▸ Accessibility (WCAG 2 AA)
     <count> serious violations — <brief list>
     None → write "Pass"
   ```

   These are the minimum quality bar for a new screen before merge — not optional. Screens
   being re-accepted after a UI bug fix skip this step unless QA asks.

   Measurements depend on network/machine — note "single measurement"; if near the warning
   threshold but not clearly over, suggest re-measuring instead of calling it a bug.

6. Write the report using the template below.

## Multi-screen feature (has a "Screens in flow" table in requirements)

Repeat steps 2–5 for **each row** in the table — each screen gets one Figma MCP call + one
real URL opened. Figma links for different screens may come from different files (different
file-key in the URL); that's fine — parse node-id from each link independently.

Save screenshots per screen: `reports/figma-check/<feature>/<screen-name>-{desktop,tablet,mobile}.png`.
The report has one section per screen (see multi-screen template below), with a total
summary at the end.

No Figma link for a specific screen → write "Figma has no design for this screen", skip
that screen, don't hold up the whole set.

## Report template

```
UI acceptance: <screen name> · <domain>
Figma: <frame link>

▸ Serious issues (<n>)
  - [desktop] Missing "Apply discount code" button — in Figma, not on real screen
  - [mobile 375] Heading text overflows the container, last word cut off

▸ Minor issues (<n>)
  - [desktop] Button text: Figma "Pay now" / real screen "Pay"

▸ Matching: <items checked and correct, one brief line>

Screenshots: <paths to 3 images>

Suggested assignment: FE — <one-sentence summary>

## Needs clarification
- <anything that couldn't be checked> — <reason>
```

Severity guide:
- **Serious**: missing/extra element, broken function, broken layout, text cut off, button
  untappable.
- **Minor**: small text mismatch, visible spacing/colour deviation that doesn't block use.
- Not sure → classify as **serious** and note it, let a human decide.

Multi-screen: repeat the above block for each screen (change the heading to
`Screen <n>: <screen name>`), concatenated in one report, with a total at the end:
`Total: <n> serious, <m> minor across <k> screens`.

## Never do this

- **Never fix the UI code.** This skill's job is acceptance and reporting only.
- **Never write "matches design"** for parts you haven't actually compared — write
  "could not verify" with the reason.
- **Never decide which side is wrong** when Figma and the real screen differ without a clear
  reason (Figma may be outdated, or there may be an agreed design change) → add to "Needs
  clarification".
- Don't compare pixel-by-pixel. Discrepancies below visible threshold → skip, don't create
  noise for dev.
