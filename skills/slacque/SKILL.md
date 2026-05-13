---
name: slacque
description: Use when generating a Slack theme from a logo or brand image. Produces 4 hex codes ready to paste into Slack's custom theme import (Preferences → Themes → Customize → Import).
---

# Slacque — Slack Theme Designer

You generate a custom Slack theme from a logo image. Slack accepts a string of 4 hex codes (`#RRGGBB, #RRGGBB, #RRGGBB, #RRGGBB`) and derives the rest of the UI from it.

## The 4 Slack slots

The string is consumed by Slack in this order:

| # | Slot | What Slack does with it |
|---|---|---|
| 1 | System navigation | Drives the sidebar — the dominant brand surface |
| 2 | Window background | Even darker — background layers / structural contrast |
| 3 | Presence indication (source) | "Online" green — Slack derives a lighter shade |
| 4 | Notifications / Selected items (source) | Mid-saturated accent — drives badges and light highlights |

**Output format:** 4 hex codes, separated by `, ` (comma + space), single line, no trailing punctuation.

Example: `#611F69, #39063A, #20A271, #C474D3`

## Step 1: Extract candidate colors from the logo

Look at the logo using vision. Identify candidate colors using these rules:

- Ignore near-white, near-black, and desaturated gray pixels (those are typically background, text, shadows — not the brand)
- Among the remaining saturated pixels, identify the **dominant hue** — that's the brand primary color
- If a **second distinct saturated hue** is present, note it as an accent candidate (it will feed slot #4)
- On gradients, take the central hue, not the extremes
- If the logo is monochrome (one saturated color), use the monochrome derivation rules in Step 4

## Step 2: Optional brand-book lookup

After the initial analysis, try to **identify the brand** from the logo (visible text, distinctive shape, recognizable monogram).

If the brand is plausibly recognized, ask the user:

> "I think this logo is from **[brand name]**. Can I check their brand site (e.g., `brand.[brand].com`, `[brand].com/brand`, `design.[brand].com`) for official colors? (yes/no)"

**If the user says yes:**
1. Use `WebSearch` for queries like `"[brand] brand guidelines colors"` or `"[brand] brand book hex"`
2. Use `WebFetch` on the most promising URLs returned
3. Extract official hex codes from the page content
4. If you find official colors, use them as the source of truth for the palette
5. If the search/fetch fails (no brand site found, or no extractable hex codes), fall back silently to vision-only and mention: *"Couldn't find a public brand book, relying on the logo alone."*

**If the user says no, or the brand is not identified:** proceed directly to vision-only extraction.

## Step 3: Construct the 4-slot palette

Starting from the brand primary color (vision) or official colors (brand book):

- **Slot #1 (System navigation):** brand color in a deep/saturated form. If the brand is already dark, take it as-is; otherwise, darken it while preserving saturation.
- **Slot #2 (Window background):** an even darker version of #1 (same hue or a neighboring hue). Must be perceptibly darker than #1.
- **Slot #3 (Presence indication):**
  - Default: a saturated green close to Slack's native green (around `#20A271`)
  - If the brand has a secondary green, use it instead
  - Keep this slot green to respect the "online = green" convention
- **Slot #4 (Notifications / Selected items):** brand secondary accent, mid-light, saturated. This is what will pop in badges. If the brand has a second saturated color, use it; otherwise, derive via complement (+180° hue) or triad (+120° hue) of #1.

## Step 4: Monochrome logo derivation

If the logo has only one saturated color:

- #1 = the extracted saturated color
- #2 = a darker version of #1 (lightness reduced by 30-50%)
- #3 = default presence green (around `#20A271`)
- #4 = complement (+180° hue rotation) or triad (+120° hue rotation) of #1, mid-light tone

## Step 5: Validate the palette

Before producing output, check:
- Exactly 4 hex codes in `#RRGGBB` format (case-insensitive)
- The 4 colors are not strictly identical to each other

Do **not** check WCAG contrast — Slack adapts contrast internally.

## Step 6: Produce output

Output **exactly** this format. Re-emit it in full on every refinement (no partial diffs):

`````markdown
## Slack theme for [brand or short description]

```
#xxx, #xxx, #xxx, #xxx
```

| # | Slot | Hex | Note |
|---|---|---|---|
| 1 | System navigation | `#xxx` | short description |
| 2 | Window background | `#xxx` | short description |
| 3 | Presence indication | `#xxx` | short description |
| 4 | Notifications | `#xxx` | short description |

*Rationale*: 1-2 sentences on the logo reading (or brand-book source) and structural choices.
`````

## Refinement loop

After the first palette, the user may iterate with natural-language commands. Recognize these and **re-emit the full output (string + table + rationale) every time**. No partial diffs.

| User says | You do |
|---|---|
| "darker" / "lighter" | Shift the lightness of slots #1 and #2 |
| "less saturated" / "more saturated" | Adjust saturation, especially on #4 |
| "change the accent" / "warmer/cooler badge" | Re-pick slot #4 |
| "another angle" / "try again" | Re-read the logo, interpret the brand color differently |
| "light theme" / "dark theme" | Invert the structural strategy (light #1/#2 instead of dark, etc.) |
| "keep everything but the mention" | Keep #1, #2, #3 — change only #4 |

## Edge cases

- **No logo provided** (no path, no attached image): ask the user for the source (drag-drop, paste, or path)
- **Path given but file doesn't exist**: flag the missing file and ask for a valid path
- **File is not a readable image**: flag it and ask for a valid image
