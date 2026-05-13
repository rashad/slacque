# Slacque — Backlog

Ideas for what to build, polish, and ship next. Loosely prioritized: 🔥 means "do this soon, highest leverage."

## Product features

- [ ] **🔥 Multi-variant output.** Generate 2–3 palettes per logo (e.g. *Faithful*, *Bold*, *Monochrome*) and let the user pick. Dramatically reduces refinement-loop length.
- [ ] **🔥 Deterministic color extraction.** Bundle `scripts/extract.py` doing k-means clustering on actual pixels; the skill calls it instead of eyeballing hexes from a vision read. Same logo → same palette every time. Vision becomes the *interpreter* of the script's output, not the extractor.
- [ ] **🔥 Slack deep-link output.** Slack supports shareable theme URLs (`slack://customize_theme?...`). Construct one so the user clicks instead of pasting. ~10 min to investigate viability.
- [ ] **Per-slot hex override.** "Set slot 3 to `#4ABC23`" — explicit single-slot edits alongside fuzzy refinements.
- [ ] **Logo-from-URL.** Accept a URL as input, fetch the image, proceed. Removes the "where's my file" friction.
- [ ] **Slack notification preview.** ASCII or HTML block previewing what a notification badge looks like with the palette — visceral feedback in-terminal.
- [ ] **Brand-book cache.** When we find official colors for a brand, persist them under `~/.claude/plugins/data/slacque-slacque/brand-cache.json` so subsequent runs skip WebSearch.
- [ ] **Light-theme variant.** Skill mentions it as a refinement command but has no first-class light-mode logic. Formalize.
- [ ] **Color extraction from web pages.** Point at a brand's site URL, pull palette from rendered CSS rather than from a logo image. Different input modality, same output.

## Repo / discovery

- [ ] **🔥 Screenshot or short GIF in the README.** Logo → palette → actual Slack wearing it. Single biggest conversion lever — nothing else moves the needle as much as visual proof.
- [ ] **`docs/examples/` gallery.** 6–8 famous brands (Notion, Linear, Stripe, Vercel, Figma, GitHub, etc.) with their generated palettes, as a markdown table with color swatches. Doubles as smoke-test fixtures.
- [ ] **Submit to [clau.de/plugin-directory-submission](https://clau.de/plugin-directory-submission).** Land in the official Anthropic-curated marketplace. Best done after the README screenshot + examples gallery are in.
- [ ] **Pin the repo + write a launch note.** A 200-word X / LinkedIn post with the GIF outperforms the bare repo for first-week traffic.
- [ ] **French README** (`README.fr.md`). The original spec was in French — there's an audience.

## Technical hardening

- [ ] **Test fixtures.** Sample logos in `tests/fixtures/` with expected hex outputs. Regression-catches when the skill prompt is tweaked.
- [ ] **`claude plugin validate` in CI.** GitHub Action runs validate on every push. Catches manifest typos before users hit them.
- [ ] **Tag releases.** `v0.1.0` exists in the manifest but no git tag. `gh release create v0.1.0` lays the foundation for proper version bumps.
- [ ] **CHANGELOG.md.** Start one once `v0.2.0` lands.

## Future / longer horizon

- [ ] **Export to other targets.** Same logo → Discord theme, VS Code theme, terminal palette. Slacque becomes "branded theming for your dev surface area," not just Slack.
- [ ] **Brand-of-the-week.** Generate a palette for a famous brand each week and post it as social bait — also doubles the examples gallery passively.
- [ ] **Brandfetch integration.** Official brand asset API; cleaner color extraction for known brands when their site is locked down.

---

## Top 3 for the next pass

If picking the smallest set that moves the most:

1. **Screenshot/GIF in README + submit to the official directory.** Highest discovery ROI, costs an afternoon.
2. **Deterministic extraction script.** Highest *trust* ROI — turns "neat AI toy" into "I actually use this every time."
3. **Multi-variant output.** Highest *delight* ROI — fewer refinement loops, more "oh nice, I'll take the second one."
