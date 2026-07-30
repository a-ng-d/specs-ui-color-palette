# Contract — Bridge messages & analytics events

- **Status**: Draft
- **Last updated**: 2026-07-27
- **Source**: `ui-ui-color-palette/src/types/{messages,events}.ts`

Two distinct contracts, not to be confused:

1. **Messages** (`messages.ts`) — UI iframe ↔ host sandbox communication (Figma/Penpot/Sketch/Framer), consumed by the bridges ([`03-platform-bridges/bridge-actions.md`](../03-platform-bridges/bridge-actions.md)).
2. **Events** (`events.ts`) — analytics payloads (Mixpanel), one type per UI module.

## 1. Messages (UI ↔ host contract)

Common shape: `{ type: string, id: string, data: ... }` (except `NotificationMessage`, `PluginMessageData`).

| Message | `type` | `data` | Likely target bridge |
| --- | --- | --- | --- |
| `ScaleMessage` | `'UPDATE_SCALE'` | `ExchangeConfiguration` + `feature?` | `updates/updateScale.ts` |
| `ColorsMessage` | `'UPDATE_COLORS'` | `ColorConfiguration[]` | `updates/updateColors.ts` |
| `ThemesMessage` | `'UPDATE_THEMES'` | `ThemeConfiguration[]` | `updates/updateThemes.ts` |
| `SettingsMessage` | `'UPDATE_SETTINGS'` | `{ name, description, colorSpace, visionSimulationMode, algorithmVersion, textColorsTheme }` | `updates/updateSettings.ts` |
| `PaletteMessage` | `'UPDATE_PALETTE'` | `{ key, value }[]` (generic patch) | `updates/updatePalette.ts` |
| `NotificationMessage` | `'INFO'\|'SUCCESS'\|'WARNING'\|'ERROR'` | `{ message, timer? }` | — (UI display, no host action) |
| `PluginMessageData` | `string` (generic envelope) | `any` | "Box" type received on the sandbox side before dispatching to the right bridge based on `type` |

⚠️ to verify: the exact dispatch mechanism (`PluginMessageData.type` → matching bridge) isn't documented here, only inferred from naming convention — to confirm by reading `bridges/loadUI.ts` if a future development adds a new message type.

## 2. Analytics events (Mixpanel)

One event type per UI module, each with a closed `feature` union:

| Type | Module | `feature` values (summary) |
| --- | --- | --- |
| `TrialEvent` | [Modals](../02-features/modals.md) (Try Pro) | — (`date`, `trialTime`, no `feature`) |
| `PublicationEvent` | [Modals](../02-features/modals.md) (Publication) | `PUBLISH_PALETTE`, `UNPUBLISH_PALETTE`, `PUSH_PALETTE`, `PULL_PALETTE`, `REUSE_PALETTE`, `SYNC_PALETTE`, `REVERT_PALETTE`, `DETACH_PALETTE`, `ADD_PALETTE`, `SHARE_PALETTE`, `SEE_PALETTE`, `STAR_PALETTE` (+ `type?`: ORG/COMMUNITY/SELF/STARRED/UNKNOWN) |
| `ImportEvent` | Source colors | `IMPORT_COOLORS`, `IMPORT_REALTIME_COLORS`, `IMPORT_COLOUR_LOVERS`, `EXTRACT_DOMINANT_COLORS`, `CREATE_COLOR_HARMONY`, `GENERATE_AI_COLORS` |
| `ScaleEvent` | [Scale](../02-features/scale.md) | see [Palette](../01-domain-model/palette.md) §4 |
| `PreviewEvent` | [Preview](../02-features/preview.md) | `LOCK_SOURCE_COLORS`, `UPDATE_COLOR_SPACE`, `UPDATE_VISION_SIMULATION_MODE`, `DISPLAY_WCAG_SCORES`, `DISPLAY_APCA_SCORES`, `DISPLAY_WCAG_INTERVAL`, `DISPLAY_APCA_INTERVAL`, `JUMP_TO_COLOR`, `COPY_COLOR_HEX`, `OPEN_CONTRAST_REPORT` |
| `SourceColorEvent` | [Settings](../02-features/settings.md) (colors) | see [Palette](../01-domain-model/palette.md) §2 |
| `ColorThemeEvent` | [Settings](../02-features/settings.md) (themes) | see [Palette](../01-domain-model/palette.md) §3 |
| `ExportEvent` | Deploy/export | `TOKENS_NATIVE`, `TOKENS_DTCG`, `TOKENS_STYLE_DICTIONARY_V3`, `TOKENS_UNIVERSAL`, `STYLESHEET_CSS/SCSS/LESS`, `TAILWIND_V3/V4`, `APPLE_SWIFTUI/UIKIT`, `ANDROID_COMPOSE/XML`, `CSV` (+ `colorSpace?`) |
| `SettingEvent` | [Settings](../02-features/settings.md) (global) | see [Palette](../01-domain-model/palette.md) §5 |
| `ActionEvent` | [Palettes](../02-features/palettes.md) (root actions) | `CREATE_PALETTE`, `SYNC_STYLES`, `SYNC_VARIABLES`, `SYNC_TOKENS`, `GENERATE_PALETTE`, `GENERATE_PALETTE_WITH_PROPERTIES`, `GENERATE_SHEET`, `SWITCH_PALETTE`, `SWITCH_PALETTE_WITH_PROPERTIES`, `SWITCH_SHEET`, `UPDATE_DOCUMENT` (+ `colors?`, `stops?`) |
| `TourEvent` | [Modals](../02-features/modals.md) (Onboarding) | `NEXT_STEP`, `LEARN_MORE`, `END_TOUR` |
| `PricingEvent` | [Modals](../02-features/modals.md) (Pricing) | `VIEW_PRICING`, `GO_TO_PRO_WEEK/MONTH/YEAR/LIFETIME`, `GO_TO_ULTIMATE_REQUEST`, `RESET_AND_CONTINUE` |
| `LanguageEvent` | [Preferences](../02-features/preferences.md) (i18n) | — (`lang`, no `feature`) |

## Rule for any future development

Any new user action must explicitly decide:
1. whether it needs a **message** (it changes document/host state);
2. whether it should be **tracked** (a new `feature` enum value in the relevant module's event type).

The two are independent — an action can be a plain event with no message (pure UI analytics) or a message with no dedicated event (rare, but possible for very frequent updates).

## See also

- [Bridge catalog](../03-platform-bridges/bridge-actions.md) — which bridge consumes each message
- [Palette](../01-domain-model/palette.md) — the operations behind `ScaleEvent`/`SourceColorEvent`/`ColorThemeEvent`/`SettingEvent`
- Every `02-features/` spec references the events/messages relevant to it in its own "See also" section
