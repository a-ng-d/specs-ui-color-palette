# Feature — Export (code & token export)

- **Status**: Implemented (as-is behavior, with a confirmed gating bug flagged — see Acceptance criteria)
- **Package(s) concerned**: `ui-ui-color-palette`
- **UI module**: `src/ui/modes/ExportPalette.tsx`, `src/ui/contexts/Export.tsx`
- **Related spec**: [Palette](../01-domain-model/palette.md) (§8), [Color System](../01-domain-model/color-system.md), [Settings](settings.md), [Inspect](inspect.md)

## Problem

Export is one of three top-level modes (`EDIT` / `INSPECT` / `EXPORT`, switched via a tab bar) — it lets the user turn the current palette into copy-pasteable or downloadable code across roughly 20 target formats. It mostly works, but one color-space gating flag is misconfigured: OKLCH export is gated behind the exact same feature flag as LCH (a copy-paste bug), so it can never be plan-gated independently of it.

## User flow

1. From the mode switcher, the user opens Export — a separate top-level mode from Edit and Inspect, reached via the same tab bar documented in Inspect.
2. A format picker lets the user choose an export target: native/DTCG/Style Dictionary/universal design tokens; CSS/SCSS/LESS stylesheets (each with a color-space sub-choice: RGB, HEX, HSL, LCH, OKLCH, P3); Tailwind v3/v4; Apple SwiftUI/UIKit; Android Compose/XML; or CSV.
3. The generated code is shown in a read-only preview; the user can copy it to the clipboard (with a temporary "copied" confirmation) or download it as a file.
4. Downloading picks the right file name/extension per format (`.swift`, `.kt`, `.scss`, `.less`, `tailwind.config.js`, `tailwind.theme.css`, or a generic name for the rest); CSV is the only format that downloads as a **zip archive** (one file per color, grouped into per-theme folders) rather than a single file.

## Rules

- Picking a different format regenerates the code fresh from the current palette props each time (via a `getCodeFromProps` helper) — it's not a stale snapshot taken once at mount.
- Every export target and every color-space sub-choice is individually plan-gated (its own feature flag), mirroring the fine-grained per-option gating pattern also seen in Themes' vision-simulation modes.
- The OKLCH color-space option's gating flag reuses the LCH flag's feature name (`EXPORT_COLOR_SPACE_LCH` instead of `EXPORT_COLOR_SPACE_OKLCH`) — a copy-paste bug. As a result, OKLCH is always gated exactly like LCH and can't be toggled independently. *(confirmed bug — see Acceptance criteria)*
- Copy-to-clipboard uses the legacy `document.execCommand('copy')` API via a temporary offscreen textarea, with a caught-error path that shows a warning toast if it fails — not the modern async Clipboard API.
- The formats available here (`ExportEvent`/`ExportConfiguration.context`) are the same set documented as the palette's export surface in the domain model, plus the same list gains a "with semantic layer" variant when a Color System is defined — see `Palette` §8.

## Acceptance criteria

- [ ] Given the color-space picker for a CSS/SCSS/LESS export, when OKLCH is selected, then it is gated by its own `EXPORT_COLOR_SPACE_OKLCH` feature flag, independent of LCH. *(currently reuses the LCH flag — confirmed bug, fix pending)*
- [ ] Given any export format, when the user copies the code, then it is copied via the clipboard and a temporary confirmation is shown; on failure, a warning is surfaced instead of failing silently.
- [ ] Given the CSV format, when the user downloads it, then a zip archive is produced with one file per color, grouped by theme folder (default theme's colors at the archive root).
- [ ] Given any other format, when the user downloads it, then the file is saved with the correct extension for that format and the palette's name (or a translated placeholder if unnamed) as the file name.
- [ ] Given an export option the user's plan doesn't include, when they try to select it, then the action is blocked with a trial/upgrade prompt, consistent with gating elsewhere in the app.

## Out of scope

- Adding new export formats or changing which ones exist — this spec documents the current set.
- Migrating clipboard copy to the modern async Clipboard API.
- Any change to how the semantic-layer ("with Color System") export variants are generated — see [Color System](../01-domain-model/color-system.md).

## Implementation notes

Reuses the engine's `Code`/`Data` classes to turn `base`/`themes` into the selected format's content, and `ExportConfiguration` (`format`, `context`, `mimeType`, `data`) to track the current selection. `ExportPalette.tsx` owns the mode-level state (current export selection, copy/download actions); `Export.tsx` (a large, ~1700-line file) owns the format picker and per-option gating.

- **Stores**: no dedicated export store — the current export selection is local `ExportPalette` state, recomputed from the live palette props on every format change.
- **Bridges**: export is entirely client-side (no message to the host beyond the generic `POST_MESSAGE` warning toast on a failed clipboard copy); nothing is written back to the document.
- **Analytics**: every format selection is tracked via `trackExportEvent`, keyed off the clicked option's `data-value` (`ExportEvent['feature']`).
- **Credits/plan**: gating is per-format and per-color-space (see Rules); no credit consumption identified — plan-based only, same as Settings/Preview.
- **Platforms**: no platform-specific branching found in this module.

## Open questions

*(none)*

## See also

- [Palette](../01-domain-model/palette.md) §8 — the full list of export formats/contexts and their semantic-layer variants
- [Color System](../01-domain-model/color-system.md) — how the semantic layer feeds into these same export formats
- [Inspect](inspect.md) — the sibling read-only mode, reached from the same top-level mode switcher
- [Settings](settings.md) — the analogous fine-grained per-option plan gating pattern (color space, vision-simulation mode)

## History

| Date | Change |
| --- | --- |
| 2026-08-04 | Created — as-is consolidation from reading `ExportPalette.tsx` and `Export.tsx`, following up on a request to cover the missing Export/Inspect modes |
