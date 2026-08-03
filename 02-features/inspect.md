# Feature — Inspect (contrast report)

- **Status**: Implemented
- **Package(s) concerned**: `ui-ui-color-palette`
- **UI module**: `src/ui/modes/InspectPalette.tsx`
- **Related spec**: [Preview](preview.md), [Palettes](palettes.md), [Export](export.md)

## Problem

Inspect is one of three top-level modes (`EDIT` / `INSPECT` / `EXPORT`, switched via a tab bar) — a read-only counterpart to editing. Its main content is the contrast report: the exact same full-screen `ContrastReport` component used in Edit mode, reused as-is here. It works as a viewer, but the "View" row action documented in [Palettes](palettes.md) doesn't actually route here today — the palette list only has a single "Open" action, always landing in Edit mode.

## User flow

1. From the mode switcher in the top bar, the user switches to Inspect.
2. The palette's shade grid (Preview) renders exactly as in Edit mode, but with no editing affordances wired to it (add-color, add-stop, and jump-to-source-color callbacks aren't passed through, unlike Edit mode).
3. A side panel's **Report** tab shows the same full-screen contrast report used in Edit mode, opened per shade with previous/next navigation between shades.
4. Opening the Report tab for the first time auto-opens the first color's first shade.

## Rules

- The Report tab reuses the exact same `ContrastReport` component as Edit mode — same scores, same sample-text playground, same known gaps (see Preview: "report opened" over-tracking, WCAG/APCA threshold mismatch with the shade-list filters) apply identically here, since it's the same code, not a separate implementation.
- The Report tab is individually plan-gated, the same per-context gating pattern used for Edit mode's Scale/Colors/Themes/Imports/Settings tabs.
- Preview's theme switcher is available here too (reusing the same dropdown), but nothing else in Preview is editable — no add-color/add-stop entry points exist in this mode.
- The palette list's "View" action (see Palettes) doesn't currently route here — only a generic "Open" action exists on each row, which always opens the palette in Edit mode. Inspect is reachable only via the in-app mode switcher once a palette is already open.

## Acceptance criteria

- [ ] Given Inspect mode is active, when the Report tab opens for the first time, then it auto-opens the first color's first shade.
- [ ] Given the contrast-report bugs documented in Preview (report-opened over-tracking, threshold mismatch), when they're fixed there, then Inspect's Report tab inherits the fix automatically, since it's the same shared component.
- [ ] Given the palette list's "View" action is implemented to actually differ from "Open" (see Palettes' resolved open question: Open = edit, View = inspect/export), then it should route here, into Inspect mode. *(currently no such routing exists — confirmed gap)*

## Out of scope

- The Properties tab (a separate, read-only summary of the palette's name/description/preset/color space/colors/themes) that also lives in Inspect mode alongside Report — not covered here; candidate for its own spec.
- Wiring the palette list's "Open"/"View" row actions to actually launch different modes — tracked as an open item in [Palettes](palettes.md), not resolved here.
- Any change to the contrast report itself — fully owned by [Preview](preview.md).

## Implementation notes

Reuses the engine's read model (`PresetConfiguration`, `ColorConfiguration`, `ThemeConfiguration`, etc.) purely for display — no update messages originate from this mode itself.

- **Stores**: reads the same palette/themes stores as Edit and Export; writes nothing back.
- **Bridges**: no dedicated bridge — Inspect mode issues no `UPDATE_*` messages; the only outbound message class is the generic `STOP_LOADER` handler shared with Edit/Export for host-driven loading-state resets.
- **Analytics**: no Inspect-mode-specific tracking identified beyond what the shared `ContrastReport`/`Preview` components already track (see Preview).
- **Platforms**: no platform-specific branching found in this module.

## Open questions

*(none)*

## See also

- [Preview](preview.md) — the shade grid and full-screen contrast report, reused as-is by this mode
- [Palettes](palettes.md) — the "Open" vs. "View" row-action distinction this mode is the eventual target of
- [Export](export.md) — the sibling top-level mode, reached from the same mode switcher

## History

| Date | Change |
| --- | --- |
| 2026-08-04 | Created — as-is consolidation from reading `InspectPalette.tsx` and `Properties.tsx`, following up on a request to cover the missing Export/Inspect modes |
| 2026-08-04 | Recentered on the contrast report (Report tab) only; the Properties tab moved out of scope, candidate for its own spec |
