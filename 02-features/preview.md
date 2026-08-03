# Feature — Preview (preview & contrast report)

- **Status**: Draft (as-is behavior, with two confirmed bugs flagged — see Acceptance criteria)
- **Package(s) concerned**: `ui-ui-color-palette`
- **UI module**: `src/ui/modules/preview`
- **Related spec**: [Palette](../01-domain-model/palette.md), [User context](../01-domain-model/user-context.md)

## Problem

The contrast report has two confirmed bugs: its pass/fail scores can disagree with the scores used to filter shades elsewhere, and its "report opened" analytics event fires far more often than it should. This spec documents current preview behavior and marks both for fixing.

## User flow

1. From the shade list, the user can show/hide WCAG and APCA scores, and show/hide "pass range" banners for each, via independent toggles.
2. The user can filter shades by pass/fail status, separately for WCAG and APCA, and separately for light and dark foreground — a "Reset" option appears once any filter is active.
3. Clicking a shade opens a full-screen contrast report (while editing) showing:
   - The source color and shade name, with previous/next navigation between shades.
   - An editable sample-text playground (text + font weight) that resets every time the report reopens.
   - A light-text/dark-text toggle.
   - A grid of scores: WCAG ratio, APCA score, a custom "readability" score, and a minimum recommended font size for the current weight.
4. If the report itself is plan-blocked, placeholder colors are shown behind an upsell message instead of real data.
5. In the preview panel's settings (edit mode only), the user can change color space, lock/unlock source colors, switch themes, and insert a new color or stop.

## Rules

- Score filters are not persisted and not tracked — only the display toggles are.
- There's no configurable pass/fail threshold in the UI; thresholds are hardcoded, aligned with the WCAG 2.1 (ratio-based) and WCAG 3.0 (APCA-based) standards.
- The pass/fail thresholds used in the contrast report and the ones used to filter the shade list are supposed to be identical, and currently aren't (see Acceptance criteria).
- The vision-simulation control does **not** live in the preview settings panel — it lives in the themes panel instead, applied per theme.
- Plan-blocked report access shows placeholder colors, never the real palette data, behind an upsell.

## Acceptance criteria

- [ ] Given a shade's WCAG/APCA scores, when they're shown in the contrast report and when they're used to filter the shade list, then both use the identical pass/fail comparison. *(currently the two disagree — confirmed bug, fix pending)*
- [ ] Given the contrast report is open, when the user types sample text, changes font weight, or switches tabs, then no additional "report opened" analytics event fires. *(currently fires on every such re-render — confirmed bug, fix pending)*
- [ ] Given the contrast report is closed and reopened, when it opens, then exactly one "report opened" analytics event fires.
- [ ] Given a score-display toggle is flipped, when the change is applied, then it persists across sessions and is tracked in analytics.
- [ ] Given a score filter is changed, when shades are filtered, then the result updates immediately without any persistence or analytics side effect.
- [ ] Given the `REPORT` feature is plan-blocked, when a user opens a shade, then placeholder colors and an upsell message are shown instead of the real contrast data.

## Out of scope

- Making pass/fail thresholds configurable in the UI.
- Validating the "readability" score formula against UX research — it's a custom, unvalidated calculation today: the APCA (Lc) score rescaled to 100, multiplied by a font-weight weighting (1× at weight 400, 1.15× above 500, 0.85× below 400).
- Adding a vision-simulation control to the preview settings panel (it stays theme-scoped, in the themes panel).

## Implementation notes

Score display state is 4 independent booleans plus a 4-way filter state (WCAG/APCA × light/dark). The contrast report keeps its own local, non-persisted state (sample text, text-theme choice, font weight). Preview settings pass through color space, source-color lock, themes, and an update callback. Not duplicated further here — see `../01-domain-model/palette.md`.

- **Stores**: the 4 display toggles and per-shade contrast scores are read/written by sibling modules (the shade list and the preview shell), not directly by the files documented here. Color-space, lock, and vision-mode changes are applied by the preview shell, not by the sub-components themselves.
- **Vision simulation**: the vision-simulation action still present in the preview shell's update handler is legacy code — it's not currently wired to any control in preview (see Rules: the live control lives in the themes panel instead), but it was kept because it may be reconnected as a preview-scoped toggle in the future.
- **Bridges**: display-toggle changes persist via the generic item-storage message; lock/color-space/vision-mode changes persist via the generic palette-update message; blocked features trigger a trial/upgrade prompt. No "jump to this shade on the canvas" bridge exists — previous/next navigation in the contrast report only changes what's shown in the panel.
- **Analytics**: score-display toggles, opening the contrast report, color-space changes, and source-color locking are all tracked. Score filters are not tracked at all.
- **Credits**: no credit consumption identified — gating here is plan-based only.
- **Platforms**: the only platform-specific behavior found is a background color choice (for the upsell overlay) that varies by host editor.

## Open questions

- Is the lack of tracking on score filters (unlike the display toggles) intentional?

## See also

- [Palette](../01-domain-model/palette.md) — shades and their pre-computed `textContrast` (WCAG/APCA), the source of what this module displays
- [Scale](scale.md) — the contrast-ratio editing mode shares the same WCAG math as this module's report
- [Settings](settings.md) — color space and vision-simulation controls also appear in the preview panel
- [Inspect](inspect.md) — reuses this module's shade grid and contrast report verbatim in a read-only mode

## History

| Date | Change |
| --- | --- |
| 2026-07-27 | Created — as-is consolidation from an agent's code reading |
| 2026-07-30 | Rewritten at a functional level (behavior/edge cases instead of file/line references), internal links added |
| 2026-08-03 | Reformatted to the Problem/User flow/Rules/Acceptance criteria template |
| 2026-08-04 | Resolved three open questions: vision-simulation handler is legacy code kept as a possible future re-connection point; thresholds confirmed aligned with WCAG 2.1/3.0; readability-score formula documented (APCA Lc rescaled to 100 × font-weight weighting) |
