# Bridge catalog

- **Status**: Draft
- **Last updated**: 2026-07-30
- **Source**: `ui-ui-color-palette/src/bridges/`

*Bridges* are the bridge between the UI (Preact iframe) and the design tool's plugin sandbox (code running with Figma/Penpot/Sketch/Framer permissions). The UI sends a typed message (see [`../04-contracts/events-messages.md`](../04-contracts/events-messages.md)), the corresponding bridge executes it on the host side, potentially via the platform's API and/or a call to `api-ui-color-palette`.

## Inventory

| Category  | File                                        | Likely role                                                                                        |
| --------- | ------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| Entry     | `loadUI.ts`                                 | Entry point: orchestrates the checks  (`checks/`) and updates.                                     |
| Checks    | `checks/checkAnnouncementsStatus.ts`        | Determines whether an announcement should be shown.                                                |
| Checks    | `checks/checkCredits.ts`                    | Checks the user's credit balance before a billed action.                                           |
| Checks    | `checks/checkEditorType.ts`                 | Resolves the current `Editor`.                                                                     |
| Checks    | `checks/checkTrialStatus.ts`                | Resolves the trial status.                                                                         |
| Checks    | `checks/checkUserConsent.ts`                | Checks consent.                                                                                    |
| Checks    | `checks/checkUserLicense.ts`                | Checks license status / triggers a `LicenseTrigger`.                                               |
| Checks    | `checks/checkUserPreferences.ts`            | Loads user preferences.                                                                            |
| Creations | `creations/createPalette.ts`                | Creates a local palette.                                                                           |
| Creations | `creations/createPaletteFromDuplication.ts` | Duplicates an existing palette.                                                                    |
| Creations | `creations/createPaletteFromRemote.ts`      | Creates a local palette from a published one (pull/reuse).                                         |
| Reads     | `gets/getPalettesOnCurrentPage.ts`          | Lists local palettes on the current page/file.                                                     |
| Reads     | `gets/jumpToPalette.ts`                     | Navigates to/selects a palette on the canvas.                                                      |
| Deletion  | `deletions/deletePalette.ts`                | Deletes a local palette.                                                                           |
| Plans     | `plans/enableTrial.ts`                      | Starts a trial.                                                                                    |
| Updates   | `updates/updateColors.ts`                   | Applies `SourceColorEvent` operations.                                                             |
| Updates   | `updates/updatePalette.ts`                  | Generic palette update (exact scope vs. `updateSettings`/`updateColors`/`updateThemes` to verify). |
| Updates   | `updates/updateScale.ts`                    | Applies `ScaleEvent` operations.                                                                   |
| Updates   | `updates/updateSettings.ts`                 | Applies `SettingEvent` operations.                                                                 |
| Updates   | `updates/updateThemes.ts`                   | Applies `ColorThemeEvent` operations.                                                              |

## Observed convention

Each bridge appears to correspond to a message `type` (`PluginMessageData.type`, e.g. `'UPDATE_SCALE'`, `'UPDATE_COLORS'`) received on the sandbox side from the UI iframe — see [`../04-contracts/events-messages.md`](../04-contracts/events-messages.md) for payload details.

## Platform-specific creation bridges (not in this catalog)

Each design-tool host repo also owns a few bridges of its own, outside this shared catalog — syncing the palette to the platform's variables/tokens, to its styles, and generating the on-canvas documentation board. They behave differently enough per platform (theme/mode support, self-documentation, where the palette is stored) to be documented separately: see [Figma](figma.md), [Penpot](penpot.md), [Sketch](sketch.md), [Framer](framer.md).

## To do when needed

- Document the exact content of each bridge (today only the file name is known) — do this in a feature spec if a future development modifies one of them.
- Clarify `updatePalette.ts` vs. the more specific updates.

## See also

- [Palette](../01-domain-model/palette.md) §2–§5 — the operations (`SourceColorEvent`, `ColorThemeEvent`, `ScaleEvent`, `SettingEvent`) each update bridge applies
- [Bridge messages & analytics events](../04-contracts/events-messages.md) — the message contract these bridges consume
- [Figma](figma.md), [Penpot](penpot.md), [Sketch](sketch.md), [Framer](framer.md) — platform-specific bridges not in this shared catalog
