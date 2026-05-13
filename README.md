# Slacque

Generate a custom Slack theme from a logo image. Produces a 4-hex string ready to paste into Slack's custom theme import.

## Install

Register the repo as a local marketplace, then install:

```
/plugin marketplace add /path/to/slacque
/plugin install slacque@slacque
```

Or use the Claude Code plugin menu (`/plugin`) after running the `marketplace add` step.

## Usage

```
/slacque ~/Downloads/acme-logo.png
```

You can also drag-and-drop the logo into the terminal after typing `/slacque ` (with a trailing space), or paste an image (Cmd+V) after `/slacque` with no argument.

After the first output, you can refine conversationally:

- "darker" / "lighter"
- "less saturated" / "more saturated"
- "change the accent" / "warmer badge" / "cooler badge"
- "another angle" / "try again"
- "light theme" / "dark theme"
- "keep everything but the mention"

For recognizable brands, Slacque will offer to check the brand's website for official colors before producing the palette (requires `WebSearch` and `WebFetch` permissions in Claude Code).

## Slack theme format

Slack accepts 4 hex codes in Preferences → Themes → Customize Your Theme → Import:

| # | Slot | Role |
|---|---|---|
| 1 | System navigation | Sidebar dominant color |
| 2 | Window background | Deeper background layer |
| 3 | Presence indication | "Online" green source (Slack derives a lighter shade) |
| 4 | Notifications | Badge and selected-item accent (Slack derives lighter shades) |

Example string: `#611F69, #39063A, #20A271, #C474D3`
