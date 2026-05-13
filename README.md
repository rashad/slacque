# Slacque

*Custom Slack themes, generated from any logo, inside Claude Code.*

Drop in a logo, get back the 4-hex string Slack expects in **Preferences → Themes → Customize Your Theme → Import**. Slacque reads the dominant brand colors with vision, optionally cross-checks the brand's website for official hex codes, and assembles a palette that fits Slack's 4-slot model. Refine it conversationally until it feels right.

## Example

```
#0E8074, #1E2D8C, #20A271, #3B4FE0
```

| # | Slot | Hex | Note |
|---|---|---|---|
| 1 | System navigation | `#0E8074` | Deepened brand teal — sidebar dominant surface |
| 2 | Window background | `#1E2D8C` | The brand's secondary navy — darker, structural layer |
| 3 | Presence indication | `#20A271` | Online-green convention |
| 4 | Notifications | `#3B4FE0` | Brighter cobalt derived from the navy — pops on badges |

*(Generated from a logo with a teal wordmark and navy accent.)*

## Install

```
/plugin marketplace add rashad/slacque
/plugin install slacque@slacque
```

Then reload (`/reload-plugins`) and run `/slacque`.

<details>
<summary>Install from a local clone (dev)</summary>

```
git clone git@github.com:rashad/slacque.git
/plugin marketplace add /path/to/slacque
/plugin install slacque@slacque
```

</details>

## Usage

```
/slacque ~/Downloads/acme-logo.png
```

You can also drag-and-drop the logo into the terminal after typing `/slacque ` (with a trailing space), or paste an image (Cmd+V) after `/slacque` with no argument.

For recognizable brands, Slacque will offer to check the brand's website for official colors before producing the palette (requires `WebSearch` and `WebFetch` permissions in Claude Code).

## Refining

After the first output, iterate in plain English:

- "darker" / "lighter"
- "less saturated" / "more saturated"
- "change the accent" / "warmer badge" / "cooler badge"
- "another angle" / "try again"
- "light theme" / "dark theme"
- "keep everything but the mention"

Each refinement re-emits the full palette — no partial diffs.

## How the 4 slots work

Slack accepts 4 hex codes and derives the rest of the UI from them:

| # | Slot | Role |
|---|---|---|
| 1 | System navigation | Sidebar dominant color |
| 2 | Window background | Deeper background layer |
| 3 | Presence indication | "Online" green source (Slack derives a lighter shade) |
| 4 | Notifications | Badge and selected-item accent (Slack derives lighter shades) |

## About

Built as a [Claude Code](https://docs.claude.com/en/docs/claude-code) plugin — markdown-driven, no external services, vision and optional web lookup come from Claude itself.

## License

MIT — see [LICENSE](./LICENSE).
