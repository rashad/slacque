# Slacque — Design Spec

**Date**: 2026-05-13
**Author**: Rashad Karanouh (with Claude)
**Status**: Implemented — see the [README](../../../README.md) for the current install and usage flow. This document is preserved as a snapshot of the original design.

## 1. Context and goal

Every time you join a new Slack workspace, the default colors are dull. The goal is a tool that, given a logo, produces a custom Slack theme ready to paste into Preferences → Themes → Customize → Import.

Slack accepts a string of **4 hex codes** in the format `#RRGGBB, #RRGGBB, #RRGGBB, #RRGGBB` and adapts contrast internally ("We'll adapt the colors as best we can to preserve contrast"). So we don't need to nail a perfect functional mapping — we just need to supply 4 coherent colors that Slack can build from.

## 2. Form factor

**Claude Code plugin**, structured around a slash command plus a skill. No executable code, no dependencies: all the "logic" lives in the prompts that guide Claude (native vision + optional web search).

### 2.1 Why not a standalone CLI

- Claude already has native vision — no need to bundle a model or call an external vision API
- Identifying the brand color in a logo requires judgment (ignore the white background, the black text, pick the dominant saturated hue) — exactly what Claude is good at
- The optional brand-book lookup (cf. §6.2) leverages `WebSearch` / `WebFetch`, which are at hand inside Claude Code

### 2.2 Why slash command + skill rather than one of them alone

- Slash command alone: forces all the knowledge into the command file, not reusable from a regular conversation
- Skill alone: no explicit entry point, less discoverable
- Both together: the command is a thin trigger that invokes the skill; the skill can also auto-activate on "generate a Slack theme from this logo" in free conversation. It's the idiomatic Claude Code plugin pattern.

## 3. Plugin layout

```
slacque/
├── .claude-plugin/
│   └── plugin.json              # manifest (name, version, description)
├── commands/
│   └── slacque.md               # thin slash command: /slacque <path>
├── skills/
│   └── slacque/
│       └── SKILL.md             # full knowledge (extraction + slots + template + refinement)
├── docs/
│   └── superpowers/specs/
│       └── 2026-05-13-slacque-design.md  # this document
└── README.md
```

The current directory is named `slack-colors-designer/` — to be renamed to `slacque/` during the first implementation phase (or not; purely cosmetic at the FS level).

Installation: `/plugin install /path/to/slacque` (or via the Claude Code plugin menu).

## 4. The `/slacque` slash command

File `commands/slacque.md`. Its only role: pick up the logo and delegate to the skill.

```markdown
---
description: Generate a Slack theme from a logo
argument-hint: <logo path> (or paste/drag the image)
---

Generate a custom Slack theme from the provided logo.

Logo source:
- If $ARGUMENTS contains a file path, read that image
- Otherwise, if an image is attached to this message, use it
- Otherwise, ask the user to provide the logo (drag-and-drop, paste, or path)

Once the logo is identified, use the `slacque` skill.
```

Supported behaviors:
- `/slacque ~/Downloads/acme-logo.png` — explicit path
- `/slacque ` then drag-and-drop an image — drag inserts the path, argument is filled
- `/slacque` + Cmd+V of a screenshot — image is attached to the message, no argument
- `/slacque` alone — Claude asks for the source

## 5. The Slack output contract (4 slots)

Decoded by cross-referencing the string `#611F69, #39063A, #20A271, #C474D3` with the chips in the Slack UI:

| # | Slot | Role in Slack rendering |
|---|---|---|
| 1 | **System navigation** | Deep brand primary — dominates the sidebar |
| 2 | **Window background** | Even darker — background layers / structural contrast |
| 3 | **Presence indication (source)** | "Online" green — Slack derives a lighter shade |
| 4 | **Notifications / Selected items (source)** | Mid-saturated secondary accent — Slack derives badges and light highlights |

**String format**: 4 hex codes in upper or lower case, separated by `, ` (comma + space), no newline.

Example: `#611F69, #39063A, #20A271, #C474D3`

## 6. The `slacque` skill

File `skills/slacque/SKILL.md`. The heart of the plugin. Four blocks:

### 6.1 Extraction from the logo (vision)

Rules the skill imposes on Claude:
- Ignore near-white, near-black, and desaturated gray pixels (background, text, shadows — not the brand)
- Among the remaining saturated pixels, identify the **dominant hue** = brand primary color
- If the logo contains a **second distinct saturated hue**, note it as accent candidate (feeds slot #4)
- On gradients, take the central hue (not the extremes)
- If the logo is monochrome, derive the other slots algorithmically (cf. §6.4)

### 6.2 Optional brand-book lookup (Option 3 — propose & confirm)

After the initial vision analysis, Claude tries to **identify the brand** (text in the logo, distinctive shape, recognizable monogram). If a brand is plausibly recognized:

> "I think this logo is from **\<brand\>**. Can I check their brand site (e.g. `brand.<brand>.com`, `<brand>.com/brand`, `design.<brand>.com`) for official colors? (yes/no)"

If **yes**:
1. `WebSearch` for e.g. `"<brand> brand guidelines colors"` or `"<brand> brand book hex"`
2. `WebFetch` on promising URLs
3. Extract official hex codes visible on the page
4. On success → use those hex codes as source of truth for the palette
5. On failure (no brand site found, or no extractable hex codes) → silent fallback to vision-only with the note: *"Couldn't find a public brand book, relying on the logo alone."*

If **no** or brand not identified → straight to vision-only.

### 6.3 Palette construction (4 slots)

Starting from the brand primary color (vision) or the official colors (brand site):

- **Slot #1 (System navigation)**: brand color in a deep/saturated form. If the brand is already dark, take it as-is; otherwise, darken while preserving saturation.
- **Slot #2 (Window background)**: an even darker version of #1 (same hue or a neighboring hue). Must be perceptibly darker than #1.
- **Slot #3 (Presence indication)**:
  - By default, a saturated green close to Slack's native green
  - If the brand has a secondary green, use it instead
  - Keep the slot green to respect the "online = green" convention
- **Slot #4 (Notifications / Selected items)**: brand secondary accent, mid-light, saturated. This is what will pop in badges. If the brand has a second saturated color, use it; otherwise, derive via complement or triad of #1.

### 6.4 Monochrome logo case (single saturated color)

- #1 = extracted saturated color
- #2 = darkened version of #1 (-30 to -50% lightness)
- #3 = default presence green (close to Slack's native green)
- #4 = complement (+180° hue rotation) or triad (+120° hue rotation) of #1, mid-light tone

### 6.5 Output template

Exact format the skill imposes on Claude after each analysis or refinement:

````markdown
## Slack theme for [brand or description]

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
````

### 6.6 Refinement loop

After the first palette, the skill instructs Claude to recognize natural-language refinement commands:

| You say | Claude does |
|---|---|
| "darker" / "lighter" | Shifts the lightness of slots #1 and #2 |
| "less saturated" / "more saturated" | Adjusts saturation, especially on #4 |
| "change the accent" / "warmer/cooler badge" | Re-picks slot #4 |
| "another angle" / "try again" | Re-reads the logo, interprets the brand color differently |
| "light theme" / "dark theme" | Inverts the structural strategy |
| "keep everything but the mention" | Keeps #1, #2, #3, changes only #4 |

On every iteration, Claude **re-emits the full string + the full table** in the §6.5 format. No partial diffs.

## 7. Validation

**Pre-analysis**:
- No logo (no argument, no attached image) → Claude asks
- Path given but file doesn't exist → Claude flags and asks
- File isn't a readable image → Claude flags

**Post-production**:
- The string contains exactly 4 hex codes in `#RRGGBB` format (case-insensitive)
- The 4 colors are not strictly identical to each other
- **No** WCAG check — Slack adapts contrast internally

## 8. Non-goals (explicitly out of MVP scope)

- URL input (website or image URL) — the user downloads the image locally first
- Multiple variants (dark / light / vibrant) produced in parallel — one palette per run, conversational refinement
- Visual mockup of the Slack sidebar — the verification table is enough
- Distribution via a public marketplace — local install only to start
- Automated build / tests — this is markdown guiding an LLM, not code to test

## 9. Plugin manifest

Minimal `.claude-plugin/plugin.json` file:

```json
{
  "name": "slacque",
  "version": "0.1.0",
  "description": "Generate a custom Slack theme from a logo"
}
```

## 10. README

Short file at the root, containing:
- A one-liner on what the plugin does
- Install: `/plugin install /path/to/slacque`
- Standard usage: `/slacque <path>` or drag-and-drop
- Mention of the conversational refinement flow
- The format of the 4 Slack slots (so we remember six months from now)
