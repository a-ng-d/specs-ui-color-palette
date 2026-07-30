# Glossary

- **Status**: Draft
- **Last updated**: 2026-07-27

Domain terms, as used in the code (`ui-ui-color-palette`, `engine-ui-color-palette`). If a term here diverges from the code, the code wins — fix this file.

## Palette

See [`01-domain-model/palette.md`](../01-domain-model/palette.md) for the full model.

| Term | Definition |
| --- | --- |
| **Source Color** | The starting color provided by the user (`SourceColorConfiguration`): image, prompt, harmony, or manual entry. |
| **Base** | Root configuration of a palette: `name`, `colors` (source colors), `colorSpace`, `algorithmVersion`. |
| **Theme** | A variant of the palette (`ThemeConfiguration`): id, name, `scale` (lightness stops), `visionSimulationMode`, `textColorsTheme`. A palette has one or more themes (e.g. light/dark). |
| **Scale / Preset** | A named set of stops applied to a theme. Known presets: Material, Material 3, Tailwind, Ant, Bootstrap, Radix, Untitled UI, Open Color, ADS / ADS Neutral, Spectrum / Spectrum Neutral, Carbon, Base, Polaris, Fluent, or Custom (1-10, 10-100, 100-1000). See `ScaleEvent` in `types/events.ts`. |
| **Shade** | A color value generated at a given stop of a theme, with pre-computed text contrast scores (`textContrast.wcag`, `textContrast.apca`). |
| **Color Space** | The color space the palette is computed in: RGB, HSL, HSLuv, LAB, LCH, OKLCH… (`ColorSpaceConfiguration`). |
| **Algorithm Version** | Version of the palette-generation algorithm (`AlgorithmVersionConfiguration`) — versioned so existing palettes don't break when the algorithm evolves. |
| **Vision Simulation Mode** | Color-blindness simulation applied to a theme's preview. |
| **Text Colors Theme** | A pair of text colors (light/dark) associated with a theme, used for contrast scores. |
| **PaletteData** | Output of `Data.makePaletteData()` / `makePaletteFullData()` on the engine side — the fully resolved palette (themes → colors → shades). Not to be confused with `Base`/`Themes`, which are the generation *inputs*. |

## Color System (semantic layer)

See [`01-domain-model/color-system.md`](../01-domain-model/color-system.md) for the data model, and [`02-features/color-system.md`](../02-features/color-system.md) for current behavior + the not-yet-built design-tool UI.

| Term | Definition |
| --- | --- |
| **Taxonomy / Schema** | A set of `groups` (dimensions, e.g. Role, Prominence, State), each with `members` (possible values, e.g. brand/neutral). |
| **Binding** | Association of a taxonomy path (`path: ['brand', 'default']`) to a primitive shade reference (`ref: 'blue:500'`), with optional per-theme `overrides` and an `isExcluded` flag to exclude it from code generation. |
| **SystemData** | Output of `System.makeSystemData()` — the resolved semantic tokens, ready for combined code generation (primitives + semantic). |

## User context

See [`01-domain-model/user-context.md`](../01-domain-model/user-context.md) for the full model, and [`02-features/preferences.md`](../02-features/preferences.md) / [`02-features/modals.md`](../02-features/modals.md) for where it surfaces in the UI.

| Term | Definition |
| --- | --- |
| **Platform** | `figma \| penpot \| sketch \| framer` — the design tool family. |
| **Editor** | Finer-grained than Platform: `figma \| figjam \| dev \| dev_vscode \| buzz \| penpot \| sketch \| framer`. A single Platform (figma) has several possible Editors. |
| **Plan / Trial / Credits** | Commercial model: `PlanStatus` (UNPAID/PAID/NOT_SUPPORTED), `TrialStatus` (UNUSED/PENDING/EXPIRED/SUSPENDED), and a credits system consumed per action (see `config.fees` and `01-domain-model/user-context.md`). |
| **Consent** | Versioned user consent (GDPR/tracking), checked by `checkUserConsent`. |

## Published palettes

See [`01-domain-model/palette.md`](../01-domain-model/palette.md) §6, [`02-features/palettes.md`](../02-features/palettes.md) (local side), and [`02-features/modals.md`](../02-features/modals.md) (Publication modal).

| Term | Definition |
| --- | --- |
| **Local Palette** | A palette stored in the design tool's document (page/file), no account required. |
| **Remote / Published Palette** | A palette pushed to the UI Color Palette platform (Supabase). Browsing context `self \| community \| org \| starred` — not a direct choice at publish time (see [`../01-domain-model/palette.md`](../01-domain-model/palette.md) §6): self = creator = current user, community = any public share, org = managed by an organization admin on the remote side. |
| **Publish / Share / Unshare / Update / Unpublish** | Lifecycle of a published palette — see `PublicationEvent` and the corresponding [MCP tools](../04-contracts/mcp-tools.md). |

## UI navigation

| Term | Definition |
| --- | --- |
| **Service** | Root navigation tab: `MANAGE \| GEN \| EXTRACT \| WHEEL \| EXPLORE`. |
| **Subservice** | `BROWSE \| OPEN` — a sub-state of a Service. |
| **Mode** | `EDIT \| INSPECT \| EXPORT` — interaction mode on the current palette. |
| **Context** | The active panel in the UI (e.g. `SCALE`, `COLORS`, `THEMES`, `EXPORT`, `SETTINGS`, `REMOTE_PALETTES_COMMUNITY`…). |
