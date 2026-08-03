# Feature — Themes (palette themes)

- **Status**: Draft (as-is behavior, with confirmed duplication flagged — see Acceptance criteria)
- **Package(s) concerned**: `ui-ui-color-palette`
- **UI module**: `src/ui/contexts/Themes.tsx`
- **Related spec**: [Palette](../01-domain-model/palette.md) (§3), [Scale](scale.md), [Colors](colors.md), [Settings](settings.md), [Preview](preview.md)

## Problem

Themes are the mechanism behind the "active theme" concept referenced across Scale, Preview, and Settings (vision-simulation mode, text colors, and the scale itself all apply to whichever theme is currently enabled), but the module implementing themes wasn't documented yet. It mostly works, but duplicates its "add theme" logic under two entry points tracked as two different analytics events, exposes a second, independent vision-simulation-mode control that overlaps with the one in Settings, and shares the same hex-validation gap as Colors.

## User flow

1. The user sees the palette's custom themes as a sortable list; exactly one default theme always exists and is never shown/edited in this list (excluded from rendering, pinned first on reorder).
2. When no custom theme exists yet, four disabled, non-interactive placeholder rows (light, light-protanomaly, dark, dark-protanomaly) are shown purely as an illustration, with a callout inviting the user to add their first theme.
3. The user adds a theme (from the header button, or a dropdown elsewhere in the app) — up to a plan-gated limit (default 2 custom themes). The new theme becomes the sole **active** theme (every other theme is disabled), seeded from the current preset's scale, a white palette background, no vision simulation, and default white/black text colors.
4. From a theme's row, the user can rename it (24-character limit), and from its "more parameters" panel: pick a vision-simulation mode (none, or one of 8 color-blindness types) for that specific theme, set its light/dark text colors, set the palette's background color, and write a description.
5. The user removes a theme; if custom themes remain, the first one becomes active, otherwise the default theme does.
6. The user reorders custom themes by drag-and-drop — the default theme always stays pinned first regardless.

## Rules

- Exactly one theme is "active" (`isEnabled: true`) at any time; adding, removing, or reordering re-derives which theme is active.
- Whichever theme is active drives three palette-wide values read by other modules: `scale`, `visionSimulationMode`, and `textColorsTheme` — this is the actual mechanism behind "applies to the active theme only" as documented in Preview and Settings.
- Vision-simulation mode has its own per-mode plan gate (9 separate feature flags: "none" plus each of the 8 color-blindness types) — a finer granularity than a single on/off gate for the whole control.
- Vision-simulation mode is editable from **two separate places**: this module's per-theme "more parameters" panel (explicitly targets one theme by id, active or not), and a second, independent control in Settings' `ColorSettings.tsx` (implicitly targets whichever theme is active). Both dispatch through different message paths for what is ultimately the same field. *(confirmed duplication — see Acceptance criteria)*
- Adding a theme is implemented twice — once inline in this module's own handler (`ADD_THEME`), once as a standalone method invoked from a dropdown elsewhere (`ADD_THEME_FROM_DROPDOWN`) — both do the exact same thing but are tracked as two different analytics features. *(confirmed duplication — see Acceptance criteria)*
- Name is capped at 24 characters; a duplicate name gets " 2" appended once, same single-collision behavior as Colors.
- The only format validation in this module is the same hex-color regex used in Colors, applied to the palette-background and the two text-color pickers — and it has the same silent-resend gap: an invalid value still sends the update message and tracks the event, using stale data, instead of being suppressed. *(confirmed by reading `updatePaletteBackgroundColor`/`updateTextLightColor`/`updateTextDarkColor`; resolves the analogous open question previously flagged in Settings)*
- Reordering always keeps the (hidden, unlisted) default theme pinned first; only custom themes are actually reorderable.

## Acceptance criteria

- [ ] Given the theme list, when the user adds a theme, then the new theme becomes the sole active theme and the palette's scale/vision-simulation/text-colors update to match it immediately.
- [ ] Given a theme is removed, when a custom theme still exists, then the first remaining custom theme becomes active; when none does, then the default theme becomes active.
- [ ] Given the theme-add action, when triggered from either this module's own header button or the external dropdown, then it is tracked under a single, consistent analytics feature. *(currently split between `ADD_THEME` and `ADD_THEME_FROM_DROPDOWN` for identical behavior — confirmed duplication, fix pending)*
- [ ] Given vision-simulation mode is changed, when it's changed from either this module's per-theme control or Settings' control, then both go through the same code path and target the same theme unambiguously. *(currently two independent implementations — confirmed duplication, fix pending)*
- [ ] Given the palette-background or text-color pickers, when the user enters an invalid hex value, then no update message is sent and no analytics event fires. *(currently a message is sent and an event tracked regardless, using stale data — confirmed gap, same as Colors, fix pending)*
- [ ] Given the theme quota is reached, when the user tries to add a theme, then the action is blocked with a trial/upgrade prompt.
- [ ] Given the theme list is reordered, when the change is applied, then the default theme remains first regardless of the drag target.

## Out of scope

- Making the default theme visible/editable from this list — it's intentionally excluded from rendering.
- Redesigning the placeholder-rows illustration shown in the empty state.
- Any change to how the active theme's scale/vision-mode/text-colors propagate to Scale/Preview/Settings — that mechanism is documented, not being redesigned here.

## Implementation notes

Reuses the engine's `ThemeConfiguration`, `PresetConfiguration`, `ScaleConfiguration`, `TextColorsThemeConfiguration`, and `VisionSimulationModeConfiguration`. Like Colors, this lives directly under `src/ui/contexts/Themes.tsx` rather than `src/ui/modules/`.

- **Stores**: every change goes through a shared `applyThemeChanges` helper that writes the full themes array to `$themes` and, derived from whichever theme is active, also writes `scale`, `visionSimulationMode`, and `textColorsTheme` to the main palette store — this is the single place all three of those cross-module values originate from (except when set via Settings' own vision-simulation control, see Rules).
- **Bridges**: all theme edits funnel through a single `UPDATE_THEMES` message carrying the full themes array, mirroring the Colors/Scale pattern of full-payload messages.
- **Analytics**: all theme actions are tracked (`ADD_THEME` / `ADD_THEME_FROM_DROPDOWN`, `RENAME_THEME`, `UPDATE_BACKGROUND`, `UPDATE_VISION_SIMULATION_MODE`, `UPDATE_TEXT_COLORS_THEME` ×2, `DESCRIBE_THEME`, `REMOVE_THEME`, `REORDER_THEME`).
- **Platforms**: the only platform-specific behavior is a background-color choice (Figma/Penpot/Sketch/Framer CSS variables) behind the empty-state gradient overlay — same pattern as Preview's upsell background.

## Open questions

*(none)*

## See also

- [Palette](../01-domain-model/palette.md) §3 — the full list of `ColorThemeEvent` actions and the associated bridge
- [Scale](scale.md), [Preview](preview.md), [Settings](settings.md) — all three read the active theme's scale/vision-mode/text-colors set by this module; Settings additionally duplicates the vision-simulation-mode control itself
- [Colors](colors.md) — the sibling module sharing the same file location and the same hex-validation gap

## History

| Date | Change |
| --- | --- |
| 2026-08-04 | Created — as-is consolidation from reading `Themes.tsx`, resolving the "active theme" mechanism referenced across Scale/Preview/Settings |
