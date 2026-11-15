# Platform — Claude

- **Status**: Implemented (plugin 1.0.9)
- **Host repo**: `claude-ui-color-palette` (Claude Code plugin), skills mirrored from `skills-ui-color-palette`
- **Related spec**: [Architecture](../00-overview/architecture.md) §3 (the MCP path), [MCP tools](../04-contracts/mcp-tools.md), [Palette readability report](../02-features/palette-report.md)

## Objective

Bring the palette workflow into Claude: generate, audit, document, export and push a palette from a conversation, with the same engine and API as the design-tool plugins.

## What's iso vs. what's different vs. the plugins

- **Iso — generation**: palettes come from `get_palette` / `get_color_system` through the MCP server, so the shades and contrast scores are identical to the plugins'.
- **Different — no UI**: there is no `ui-ui-color-palette` panel. The experience is a guided conversation run by the `uicper` orchestrator in four phases: Source → Palette → Deploy → Manage (plus Color System between Palette and Deploy).
- **Different — output surfaces**: what the plugins show in their panel, Claude produces as conversation output or files:
  - inline swatches (markdown, or ANSI in a terminal)
  - the [readability report](../02-features/palette-report.md), the default way to see a palette
  - the `preview_palette` image, a last-resort preview
  - code files (`generate_code`) and design-tool pushes through the Figma, Penpot, Sketch and Framer MCP servers
- **Different — storage**: no document storage. The palette lives in the conversation's session slots (`PaletteData`, `SourceColors`, `SystemConfiguration`…) and on the platform once published.

## Structure

| Piece | Content |
| ----- | ------- |
| Orchestrator | `uicper`, default agent (`settings.json`) |
| Sub-agents | `palette-auditor`, `palette-codegen`, `palette-color-systemer`, `palette-publisher`, `palette-transitioner` |
| Skills | `generate-source-colors`, `scale-palette`, `audit-palette`, `report-palette`, `manage-palettes`, `build-color-system`, `generate-code`, `generate-semantic-code`, `figma`, `penpot`, `sketch`, `framer`, `help` |
| MCP servers | `ui-color-palette` (required), `figma`, `figma-desktop`, `penpot`, `sketch`, `framer`, `github`, `gitlab` |
| User config | `penpot_url`, `penpot_token` (sensitive), `framer_mcp_url`, `gitlab_url` |

## Rules

- **The skills repo is the source of truth.** Skills are edited in `skills-ui-color-palette` and copied into the plugin; the plugin never diverges.
- **One `get_palette` call per palette.** Preview, audit and report reuse the same compact rows.
- **No secrets read from the machine.** Anything sensitive is a `userConfig` option with `sensitive: true`. Repository work goes through the `github` / `gitlab` MCP servers, not through CLI tokens.
- **Bundled code only.** Scripts the plugin runs ship inside the plugin (e.g. the report renderer); nothing is downloaded and executed.

## Distribution

Published through the Yelbolt marketplace and the Anthropic plugin directory. Each pushed commit is checked by the directory (validation, directory policy, security analysis); a version that passes waits for a reviewer once publication is requested.

| Directory finding | Status in 1.0.9 |
| ----------------- | --------------- |
| Credential read from the user's machine | Removed with the vendored `gh-cli` / `gitlab-cli-skills` skills |
| Download-and-run command | Removed with the same skills |
| Unrecognized `plugin.json` fields (`documentationUrl`, `supportUrl`, `privacyPolicyUrl`, `termsOfServiceUrl`) | Kept on purpose: listing fields read by the directory only |
| Plugin changes Claude Code settings | Kept: `settings.json` sets `uicper` as the default agent |

## See also

- [Architecture](../00-overview/architecture.md) — package map and the MCP path
- [MCP tools](../04-contracts/mcp-tools.md) — the tools the plugin calls
- [Palette readability report](../02-features/palette-report.md) — the Claude-only document output
- [Web App](web-app.md) — the other surface without design-tool sync

## History

| Date | Change |
| ---- | ------ |
| 2026-11-16 | Created: platform layer for the Claude plugin, with the 1.0.9 readability report and the directory findings |
