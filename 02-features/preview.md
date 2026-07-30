# Feature — Preview (preview & contrast report)

- **Status**: Draft (as-is consolidation — no behavior change proposed at this stage)
- **Package(s) concerned**: `ui-ui-color-palette`
- **UI module**: `src/ui/modules/preview`
- **Related spec**: [Palette](../01-domain-model/palette.md), [User context](../01-domain-model/user-context.md)

## 1. Context

This spec documents contrast-score display/filtering on shades, the detailed per-color contrast report, and the preview panel's contextual settings (color space, source-color locking, insertion). Consolidation only, no proposed evolution.

## 2. Current behavior (as-is)

### Score display & filtering

Two menus in the preview panel, both plan-gated:

- **Display**: 4 independent toggles — show WCAG score, show APCA score, show WCAG "pass range" banner, show APCA "pass range" banner. Each is persisted and tracked independently.
- **Filter**: for WCAG and APCA separately, light and dark foreground separately, a 3-state filter (all / passing only / failing only). Filters are **not persisted and not tracked** — they only affect what's currently shown. A "Reset" affordance reappears as soon as any filter isn't set to "all". There's **no configurable pass/fail threshold** in the UI — the thresholds are hardcoded (see the report section below, and §6 for a discrepancy between the two).

Each individual toggle is separately plan-gated, prompting a trial/upgrade flow if blocked.

### Contrast report

A full-screen panel (edit-mode inspection only), opened by clicking a shade. Contains:

- A title banner with the source color and shade name, plus previous/next navigation between shades.
- An editable (not persisted) playground: sample text and font weight, reset every time the report is reopened.
- A light-text/dark-text toggle.
- A contrast grid showing: the WCAG ratio (color-coded: fail / warning / pass, with a pass badge at the standard AA threshold), the APCA score (same color-coding logic, pass badge at its own threshold), a **locally computed "readability" score** — a custom formula derived from the APCA score and adjusted by font weight, not delegated to the shared color engine — and a minimum recommended font size for the current weight.
- If the report itself is plan-blocked: placeholder colors are shown behind an upsell message instead of the real contrast data.
- The "report opened" analytics event currently fires **on every re-render** of the panel (typing in the sample text, changing weight, switching tabs) rather than only once when the report is actually opened — confirmed a real bug, not intended (see §6).

### Preview settings

Visible only while editing (except the theme switcher, which stays visible):

- A color-space picker (9 spaces: LCH, OKLCH, LAB, OKLAB, HSL, HSV, HSLUV, CMYK).
- A lock/unlock toggle for source colors.
- A theme switcher.
- An "Insert" menu (add color / add stop), quota-gated.
- **No vision-simulation control lives in this panel** — that control actually lives in the themes panel instead, applied per theme (see §6).

## 3. Proposal (to-be)

None — consolidation spec. Any future evolution of this module should start from this documented state.

## 4. Data model

Score display state is 4 independent booleans plus a 4-way filter state (WCAG/APCA × light/dark). The contrast report keeps its own local, non-persisted state (sample text, text-theme choice, font weight). Preview settings pass through color space, source-color lock, themes, and an update callback. Not duplicated further here — see `../01-domain-model/palette.md`.

## 5. Impact

- **Stores**: the 4 display toggles and per-shade contrast scores are read/written by sibling modules (the shade list and the preview shell), not directly by the 3 files documented here. Color-space, lock, and vision-mode changes are applied by the preview shell, not by the sub-components themselves.
- **Bridges**: display-toggle changes persist via the generic item-storage message; lock/color-space/vision-mode changes persist via the generic palette-update message; blocked features trigger a trial/upgrade prompt. **No "jump to this shade on the canvas" bridge exists** — previous/next navigation in the contrast report only changes what's shown in the panel, it doesn't select anything on the document.
- **Analytics**: score-display toggles, opening the contrast report (see the re-render bug above), color-space changes, and source-color locking are all tracked. Score **filters are not tracked** at all.
- **Credits**: no credit consumption identified — gating here is plan-based only.
- **Platforms**: the only platform-specific behavior found is a background color choice (for the upsell overlay) that varies by host editor.

## 6. Open questions

- The vision-simulation action exists in the preview shell's update handler but has no control anywhere in the preview settings panel — dead code, or an extension point for a future toggle directly in preview (today it only lives in the themes panel)?
- ~~Inconsistent pass/fail thresholds between the contrast report and shade-list filtering~~ (the report uses an absolute-value APCA comparison, the shade list doesn't) — **Answered (author)**: confirmed a real bug, not intended — the two are supposed to show identical pass/fail results. Needs fixing so both use the same comparison.
- Are the pass/fail thresholds themselves (4.5/7 for WCAG, 30/45/70 for APCA) aligned with a named standard (WCAG 2.1 AA/AAA, APCA Bronze/Silver/Gold), or arbitrarily chosen? Undecided.
- Has the custom "readability score" formula been validated against any UX research, or is it a placeholder? Undecided.
- ~~"Report opened" tracked on every re-render~~ — **Answered (author)**: confirmed a real bug, not intended. Fix: track it once, at the point the report actually opens, not inside the render path.
- Score filters aren't tracked at all, unlike the display toggles — intentional?

## See also

- [Palette](../01-domain-model/palette.md) — shades and their pre-computed `textContrast` (WCAG/APCA), the source of what this module displays
- [Scale](scale.md) — the contrast-ratio editing mode shares the same WCAG math as this module's report
- [Settings](settings.md) — color space and vision-simulation controls also appear in the preview panel

## 7. History

| Date | Change |
| --- | --- |
| 2026-07-27 | Created — as-is consolidation from an agent's code reading |
| 2026-07-30 | Rewritten at a functional level (behavior/edge cases instead of file/line references), internal links added |
