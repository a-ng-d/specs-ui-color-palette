# Domain model — Palette

- **Status**: Draft
- **Last updated**: 2026-07-27
- **Source**: `engine-ui-color-palette` (`Data` class), `ui-ui-color-palette/src/types/{app,messages}.ts`, `src/bridges/`, `src/stores/palette.ts` / `localPalettes.ts`

## 1. Generation inputs

```
Base
├─ name: string
├─ colors: ColorConfiguration[]      # source colors
├─ colorSpace: ColorSpaceConfiguration
└─ algorithmVersion: AlgorithmVersionConfiguration

Theme (× N per palette)
├─ id: string
├─ name: string
├─ scale: { [stopName]: lightness }   # preset or custom stops
├─ visionSimulationMode: VisionSimulationModeConfiguration
└─ textColorsTheme: TextColorsThemeConfiguration<'HEX'>
```

`new Data({ base, themes, meta })` → `.makePaletteData()` / `.makePaletteFullData()` produces the resolved palette: each theme contains its colors, each color its shades, each shade its hex value and its pre-computed `textContrast` (WCAG + APCA, light and dark).

**`base` + `themes`** is the unit of exchange with the API/MCP (`get_palette`, `generate_code`, `get_color_system` all take `base`/`themes`, not an opaque `paletteData` object) — see [`04-contracts/api-endpoints.md`](../04-contracts/api-endpoints.md).

## 2. Source color operations (`SourceColorEvent`)

`RENAME_COLOR`, `REMOVE_COLOR`, `ADD_COLOR`, `UPDATE_HEX`, `UPDATE_LCH`, `SHIFT_HUE`, `SHIFT_CHROMA`, `RESET_HUE`, `RESET_CHROMA`, `DESCRIBE_COLOR`, `SWITCH_ALPHA_MODE`, `UPDATE_BACKGROUND_COLOR`, `REORDER_COLOR`.

Associated bridge: `src/bridges/updates/updateColors.ts`.

## 3. Theme operations (`ColorThemeEvent`)

`RENAME_THEME`, `REMOVE_THEME`, `ADD_THEME`, `ADD_THEME_FROM_DROPDOWN`, `UPDATE_BACKGROUND`, `UPDATE_VISION_SIMULATION_MODE`, `UPDATE_TEXT_COLORS_THEME`, `DESCRIBE_THEME`, `REORDER_THEME`.

Associated bridge: `src/bridges/updates/updateThemes.ts`.

## 4. Scale operations (`ScaleEvent`)

Preset change (`SWITCH_MATERIAL`, `SWITCH_TAILWIND`, `SWITCH_CUSTOM[_1_10|_10_100|_100_1000]`, `SWITCH_ANT`, `SWITCH_BOOTSTRAP`, `SWITCH_RADIX`, `SWITCH_UNTITLED_UI`, `SWITCH_OPEN_COLOR`, `SWITCH_ADS[_NEUTRAL]`, `SWITCH_SPECTRUM[_NEUTRAL]`, `SWITCH_CARBON`, `SWITCH_BASE`, `SWITCH_POLARIS`, `SWITCH_FLUENT`), easing (`EasingConfiguration`, `APPLY_EASING`), `REVERSE_STOPS`, `RESET_SCALE`, contrast mode (`CONTRAST_MODE_ON/OFF`), keyboard shortcuts.

Associated bridge: `src/bridges/updates/updateScale.ts`.

## 5. Global settings (`SettingEvent`)

`RENAME_PALETTE`, `DESCRIBE_PALETTE`, `UPDATE_VIEW`, `UPDATE_COLOR_SPACE`, `UPDATE_VISION_SIMULATION_MODE`, `UPDATE_ALGORITHM`, `UPDATE_TEXT_COLORS_THEME`.

Associated bridge: `src/bridges/updates/updateSettings.ts`.

## 6. Lifecycle — local vs. published

```
Creation          bridges/creations/{createPalette, createPaletteFromDuplication, createPaletteFromRemote}.ts
Read (local)       bridges/gets/{getPalettesOnCurrentPage, jumpToPalette}.ts
Deletion           bridges/deletions/deletePalette.ts
```

A **local** palette lives inside the design tool's document (`stores/localPalettes.ts`). A **published** palette lives on the platform (Supabase, via `api-ui-color-palette`) with a visibility of `self | community | org | starred` (`Context`: `REMOTE_PALETTES_SELF|COMMUNITY|ORG|STARRED`).

Transitions between the two, captured by `PublicationEvent.feature`:
`PUBLISH_PALETTE`, `UNPUBLISH_PALETTE`, `PUSH_PALETTE`, `PULL_PALETTE`, `REUSE_PALETTE`, `SYNC_PALETTE`, `REVERT_PALETTE`, `DETACH_PALETTE`, `ADD_PALETTE`, `SHARE_PALETTE`, `SEE_PALETTE`, `STAR_PALETTE`.

The precise semantics of `PUSH` vs `SYNC` vs `REVERT` vs `DETACH` are documented by the `PublicationStatus` state machine (9 states) — see [`../02-features/modals.md`](../02-features/modals.md) (Publication section) for the full status → action(s) table.

### Visibility of a published palette (confirmed by the author, 2026-07-27)

`Publication.tsx` only exposes a **boolean `isShared`** on the publisher's side ("Share", yes/no) — `PublicationConfiguration` (engine) only contains `{isPublished, isShared}`, with no notion of scope at publish time. The self / community / org distinction (used for **browsing**, see `Context` in `00-overview/glossary.md`) is computed differently:

- The remote (Supabase, on the `api-ui-color-palette` side) carries a **publishing account type** `MEMBER | ORG` on the palette.
- **Self** = comparison `creator_id === current user` (these are my own palettes).
- **Community** = any shared palette (`isShared: true`) visible to everyone.
- **Org** = managed on the remote side by an **organization admin** (scope managed outside the client toggle — not a choice exposed in `Publication.tsx`).

⚠️ This description comes from the author, not from reading the Supabase schema / `api-ui-color-palette` code — to confirm against the API code if a future development touches the org-scoping logic.

## 7. Quotas

`Config.limits`: `localPalettes`, `sourceColors`, `customStops`, `colorThemes` — caps likely dependent on plan (Free/Pro), to confirm in a dedicated spec if they need to evolve.

## 8. Export

Formats supported by `generate_code` (primitives only): see [`04-contracts/api-endpoints.md`](../04-contracts/api-endpoints.md). The same formats exist in a "with semantic layer" version when a Color System is defined (`ExportEvent` lists the same values: `TOKENS_NATIVE`, `TOKENS_DTCG`, `TOKENS_STYLE_DICTIONARY_V3`, `TOKENS_UNIVERSAL`, `STYLESHEET_CSS/SCSS/LESS`, `TAILWIND_V3/V4`, `APPLE_SWIFTUI/UIKIT`, `ANDROID_COMPOSE/XML`, `CSV`) — see [`01-domain-model/color-system.md`](color-system.md).

## See also

- [Glossary](../00-overview/glossary.md) — Palette / Color System / Published palettes terms
- Features built on this model: [Palettes](../02-features/palettes.md), [Scale](../02-features/scale.md), [Settings](../02-features/settings.md), [Preview](../02-features/preview.md), [Modals](../02-features/modals.md) (publication lifecycle)
- How primitives sync to design tools: [`03-platform-bridges/bridge-actions.md`](../03-platform-bridges/bridge-actions.md) and the per-platform specs
- External surface: [REST API](../04-contracts/api-endpoints.md), [MCP tools](../04-contracts/mcp-tools.md), [bridge messages](../04-contracts/events-messages.md)
