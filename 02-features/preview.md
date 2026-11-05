# Feature — Preview (preview & contrast report)

- **Status**: Implemented (as-is behavior, with three confirmed bugs flagged — see Acceptance criteria)
- **Package(s) concerned**: `ui-ui-color-palette`
- **UI module**: `src/ui/modules/preview`
- **Related spec**: [Palette](../01-domain-model/palette.md), [User context](../01-domain-model/user-context.md)

## Problem

The contrast report has two confirmed bugs: its pass/fail scores can disagree with the scores used to filter shades elsewhere, and its "report opened" analytics event fires far more often than it should. This spec documents current preview behavior and marks both for fixing.

## User flow

1. From the shade list, the user can show/hide WCAG and APCA scores, and show/hide "pass range" banners for each, via independent toggles. In the full (non-compact) shade list, hovering a shade swaps its score chips to an enriched label — the WCAG 2.1 grade (A, AA or AAA) and the APCA minimum recommended usage — when the contrast report is available to the user; otherwise the chips keep the raw score (`4.52 : 1`, `Lc 61.3`) on hover. The compact shade list is unaffected.
2. The user can filter shades by pass/fail status, separately for WCAG and APCA, and separately for light and dark foreground — a "Reset" option appears once any filter is active.
3. Clicking a shade opens a full-screen contrast report (while editing) showing:
   - The source color and shade name, with previous/next navigation between shades.
   - An editable sample-text playground (text + font weight) that resets every time the report reopens.
   - A light-text/dark-text toggle.
   - A grid of scores: WCAG ratio, APCA score, a custom "readability" score, and a minimum recommended font size for the current weight.
4. In the preview panel's settings (edit mode only), the user can change color space, lock/unlock source colors, switch themes, and insert a new color or stop.

## Rules

- Score filters are not persisted and not tracked — only the display toggles are. Non-persistence is intentional (filters are meant to be a transient view, reset on reopen); the lack of tracking is **not** — confirmed a gap to fix, not deliberate (see Acceptance criteria).
- There's no configurable pass/fail threshold in the UI; thresholds are hardcoded, aligned with WCAG 2.1 (ratio-based) and with APCA, the contrast method proposed for WCAG 3.0 (still a W3C Working Draft).
- The pass/fail thresholds used in the contrast report and the ones used to filter the shade list are supposed to be identical, and currently aren't (see Acceptance criteria).
- **Score chips never resize the grid on hover.** Each chip is as wide as its content, capped at the space of its cell. When a shade is hovered, the chips are capped at the width the cell already has, so an enriched label longer than that is truncated with an ellipsis instead of widening the cell, the row and the whole grid. Hovering never changes a cell's width or height.
- The vision-simulation control does **not** live in the preview settings panel anymore — it lives in Settings (for the active theme) and in each theme's parameters, and it is always tied to a theme (color mode).

## Acceptance criteria

- [ ] Given a shade's WCAG/APCA scores, when they're shown in the contrast report and when they're used to filter the shade list, then both use the identical pass/fail comparison. *(currently the two disagree — confirmed bug, fix pending)*
- [ ] Given the contrast report is open, when the user types sample text, changes font weight, or switches tabs, then no additional "report opened" analytics event fires. *(currently fires on every such re-render — confirmed bug, fix pending)*
- [ ] Given the contrast report is closed and reopened, when it opens, then exactly one "report opened" analytics event fires.
- [ ] Given the full shade list, when the user hovers a shade, then no cell changes width or height and the grid does not shift; an enriched label that doesn't fit ends with an ellipsis.
- [ ] Given the contrast report is not available to the user, when they hover a shade, then the score chips keep their raw format.
- [ ] Given a score-display toggle is flipped, when the change is applied, then it persists across sessions and is tracked in analytics.
- [ ] Given a score filter is changed, when shades are filtered, then the result updates immediately without persistence — but **is** tracked in analytics, matching the display toggles. *(currently untracked — confirmed gap, not intentional, fix pending)*

## Out of scope

- Making pass/fail thresholds configurable in the UI.
- Validating the "readability" score formula against UX research — it's a custom, unvalidated calculation today: the APCA (Lc) score rescaled to 100, multiplied by a font-weight weighting (1× at weight 400, 1.15× above 500, 0.85× below 400).
- Adding a vision-simulation control to the preview settings panel (it stays theme-scoped, in the themes panel).

## Implementation notes

Score display state is 4 independent booleans plus a 4-way filter state (WCAG/APCA × light/dark). The contrast report keeps its own local, non-persisted state (sample text, text-theme choice, font weight). Preview settings pass through color space, source-color lock, themes, and an update callback. Not duplicated further here — see `../01-domain-model/palette.md`.

- **Stores**: the 4 display toggles and per-shade contrast scores are read/written by sibling modules (the shade list and the preview shell), not directly by the files documented here. Color-space, lock, and vision-mode changes are applied by the preview shell, not by the sub-components themselves.
- **Vision simulation**: the vision-simulation action still present in the preview shell's update handler is legacy code — it's not currently wired to any control in preview (see Rules: the live control lives in the themes panel instead), but it was kept because it may be reconnected as a preview-scoped toggle in the future.
- **Bridges**: display-toggle changes persist via the generic item-storage message; lock/color-space/vision-mode changes persist via the generic palette-update message. No "jump to this shade on the canvas" bridge exists — previous/next navigation in the contrast report only changes what's shown in the panel.
- **Analytics**: score-display toggles, opening the contrast report, color-space changes, and source-color locking are all tracked. Score filters are not tracked at all today — confirmed a gap to close, not a deliberate omission; needs a new tracked event mirroring the display-toggle one, keyed by WCAG/APCA × light/dark filter change.
- **Platforms**: no platform-specific behavior identified in these files.

## Open questions

- ~~Is the lack of tracking on score filters (unlike the display toggles) intentional?~~ — resolved (2026-08-06): no, confirmed an oversight to fix, not deliberate. See Rules and Acceptance criteria.

*(none remaining)*

## See also

- [Palette](../01-domain-model/palette.md) — shades and their pre-computed `textContrast` (WCAG/APCA), the source of what this module displays
- [Scale](scale.md) — the contrast-ratio editing mode shares the same WCAG math as this module's report
- [Settings](settings.md) — the color space control also appears in the preview panel; vision simulation does not
- [Inspect](inspect.md) — reuses this module's shade grid and contrast report verbatim in a read-only mode

## History

| Date | Change |
| --- | --- |
| 2026-07-27 | Created — as-is consolidation from an agent's code reading |
| 2026-07-30 | Rewritten at a functional level (behavior/edge cases instead of file/line references), internal links added |
| 2026-08-03 | Reformatted to the Problem/User flow/Rules/Acceptance criteria template |
| 2026-08-04 | Resolved three open questions: vision-simulation handler is legacy code kept as a possible future re-connection point; thresholds confirmed aligned with WCAG 2.1/3.0; readability-score formula documented (APCA Lc rescaled to 100 × font-weight weighting) |
| 2026-08-06 | Confirmed the missing analytics tracking on score filters is a genuine gap, not intentional — added as a third confirmed bug (status line updated), with a new acceptance criterion (filters should be tracked, mirroring the display toggles). |
| 2026-10-29 | Plan, gating and credits content removed; plans live in the docs (free-and-pro) |
| 2026-11-03 | Clarified with the product owner: vision simulation is no longer in the preview panel; it lives in Settings (active color mode) and in each theme's parameters, always tied to a mode |
| 2026-11-05 | Hover behavior of the score chips fixed (POR-668): the enriched label is tied to the contrast report's availability, and long labels are truncated with an ellipsis so the grid no longer shifts |
