# UI/Responsive check with Figma

Read when: comparing real UI against design (`figma-ui-check`).

## Extracting node-id from a QA-provided link

Figma links always follow this format: `...figma.com/design/<file-key>/...?node-id=123-456`.
The `node-id` in the URL uses a `-` separator, but when calling the tool it must be `123:456`
(colon).

- Link **has no** `?node-id=` → it's a link to the whole file, not one frame. **Ask QA which
  frame**, don't auto-pick the first or largest frame.
- Link has a `node-id` for an individual text/button (not a screen frame) → ask whether they
  really meant to compare the whole screen, to avoid missing the rest of it.

## MCP `Figma` tools — use in order, don't call them all

The server is already configured in `.mcp.json` (`https://mcp.figma.com/mcp`). Each tool
returns fairly heavy JSON/images — only call the tool you need for the current step, don't
call all of them "just to be safe".

| Tool | When to use | Returns |
|---|---|---|
| `get_metadata` | Always first | Light outline: layer tree, ids, types, positions — understand what's in the frame before diving deeper |
| `get_screenshot` | After metadata, once per node being compared | PNG preserving the layout — compare visually against a real screenshot |
| `get_design_context` | When you need exact numbers (spacing, colour, font) to document a clear discrepancy | React/Tailwind code generated from the node — read values from it, don't copy this code into the repo |
| `get_variable_defs` | When you suspect a colour/spacing mismatch but need the design-system token | List of colour/spacing/typography variables for the node |
| `search_design_system` | When the real screen uses a component whose name doesn't appear in Figma | Find matching components/styles by keyword |

Don't use the Write tool group (`use_figma`, `generate_figma_design`, ...) — this skill
only **reads** for acceptance, it never creates or modifies anything in Figma.

## Can't read the file

- No access to the file, or `node-id` is wrong (tool returns error/empty) → **state the
  reason clearly, stop, ask for a new link** — don't estimate numbers from the frame name or
  from the small link preview image.
- File is too large / deeply nested and `get_metadata` returns a very long outline → stay
  within the subtree under the given node-id; don't climb to a parent node for "context".

## Required breakpoints

Desktop 1280px, tablet 768px, mobile 375px. Figma usually only has a desktop design — when
there's no separate frame for tablet/mobile, write "Figma has no design for this size",
**don't infer** what the small screen should look like from the desktop frame.

## Number discrepancies

Any discrepancy against Figma must be noted with a reason; don't silently implement
something different. Don't decide unilaterally which side is right (Figma may be outdated,
or there may be an agreed change) — add it to "Needs clarification" in the report.
