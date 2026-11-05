# Feature — Export (code & token export)

- **Status**: Implemented (as-is behavior, with a confirmed feature-flag bug flagged — see Acceptance criteria)
- **Package(s) concerned**: `ui-ui-color-palette`
- **UI module**: `src/ui/modes/ExportPalette.tsx`, `src/ui/contexts/Export.tsx`
- **Related spec**: [Palette](../01-domain-model/palette.md) (§8), [Color System](../01-domain-model/color-system.md), [Settings](settings.md), [Inspect](inspect.md)

## Problem

Export is one of three top-level modes (`EDIT` / `INSPECT` / `EXPORT`, switched via a tab bar) — it lets the user turn the current palette into copy-pasteable or downloadable code across roughly 20 target formats. It mostly works, but one color-space feature flag is misconfigured: OKLCH export reuses the exact same feature flag as LCH (a copy-paste bug), so it can never be toggled independently of it.

## User flow

1. From the mode switcher, the user opens Export — a separate top-level mode from Edit and Inspect, reached via the same tab bar documented in Inspect.
2. A format picker lets the user choose an export target: native/DTCG/Style Dictionary/universal design tokens; CSS/SCSS/LESS stylesheets (each with a color-space sub-choice: RGB, HEX, HSL, LCH, OKLCH, plus P3 for CSS only); DTCG tokens have their own sub-choice, RGB or OKLCH; Tailwind v3/v4 (Tailwind v4 has its own sub-choice, HEX or OKLCH, available in the engine and being wired into the plugin menu); Apple SwiftUI/UIKit; Android Compose/XML; or CSV.
3. The generated code is shown in a read-only preview; the user can copy it to the clipboard (with a temporary "copied" confirmation) or download it as a file.
4. Downloading saves the file under a name that follows one convention for every format, `<palette>.<role>.<ext>` (see Rules). A format that produces several files downloads as a **zip archive** named `<palette>.zip`: the CSV (one file per color, grouped into per-theme folders) and the DTCG export with several color modes (`<mode>.primitives.tokens.json` files plus `tokens.resolver.json`).

## Rules

- Picking a different format regenerates the code fresh from the current palette props each time (via a `getCodeFromProps` helper) — it's not a stale snapshot taken once at mount.
- Every export target and every color-space sub-choice has its own feature flag.
- Each color-space option has its own feature flag, which switches the color model written in the exported code. The OKLCH color-space option's flag reuses the LCH flag's feature name (`EXPORT_COLOR_SPACE_LCH` instead of `EXPORT_COLOR_SPACE_OKLCH`) — a copy-paste bug. As a result, OKLCH always follows LCH's flag and can't be toggled independently. *(confirmed bug — see Acceptance criteria)*
- **Download file names** follow `<palette>.<role>.<ext>`, the same on every platform. `<palette>` is the palette name in kebab-case (`My Palette` → `my-palette`), or the translated placeholder name when the palette is unnamed. `<role>.<ext>` reuses the engine's file name for the format (`primitives.css`, `primitives.tokens.json`…), except for Tailwind, which keeps a `tailwind` role. The names are computed in the UI (`getExportFileName` in `ExportPalette.tsx`); the engine's own file names are unchanged and are still used for the files inside a zip.

  | Format | Downloaded file |
  | --- | --- |
  | CSS (every color space) | `<palette>.primitives.css` |
  | SCSS | `<palette>.primitives.scss` |
  | LESS | `<palette>.primitives.less` |
  | DTCG tokens, one color mode | `<palette>.primitives.tokens.json` |
  | DTCG tokens, several color modes | `<palette>.zip` |
  | Native tokens | `<palette>.tokens.json` |
  | Style Dictionary v3 | `<palette>.primitives.json` |
  | Universal JSON | `<palette>.primitives.json` |
  | Tailwind v3 | `<palette>.tailwind.config.js` |
  | Tailwind v4 | `<palette>.tailwind.css` |
  | Apple SwiftUI, UIKit | `<palette>.Primitives.swift` |
  | Android Compose | `<palette>.Primitives.kt` |
  | Android XML | `<palette>.primitives.xml` |
  | CSV | `<palette>.zip` |

- Copy-to-clipboard uses the legacy `document.execCommand('copy')` API via a temporary offscreen textarea, with a caught-error path that shows a warning toast if it fails — not the modern async Clipboard API.
- The formats available here (`ExportEvent`/`ExportConfiguration.context`) are the same set documented as the palette's export surface in the domain model, plus the same list gains a "with semantic layer" variant when a Color System is defined — see `Palette` §8.

## Acceptance criteria

- [ ] Given the color-space picker for a CSS/SCSS/LESS export, when OKLCH is selected, then it is driven by its own `EXPORT_COLOR_SPACE_OKLCH` feature flag, independent of LCH. *(currently reuses the LCH flag — confirmed bug, fix pending)*
- [ ] Given any export format, when the user copies the code, then it is copied via the clipboard and a temporary confirmation is shown; on failure, a warning is surfaced instead of failing silently.
- [ ] Given the CSV format, when the user downloads it, then a zip archive is produced with one file per color, grouped by theme folder (default theme's colors at the archive root).
- [ ] Given any other format, when the user downloads it, then the file is saved as `<palette>.<role>.<ext>` per the table in Rules, with the palette's name in kebab-case (or the translated placeholder if unnamed).
- [ ] Given a format that produces several files, when the user downloads it, then the zip archive is named `<palette>.zip` and the files inside keep the engine's names.

## Out of scope

- Adding new export formats or changing which ones exist — this spec documents the current set.
- Migrating clipboard copy to the modern async Clipboard API.
- Any change to how the semantic-layer ("with Color System") export variants are generated — see [Color System](../01-domain-model/color-system.md).

## Implementation notes

Reuses the engine's `Code`/`Data` classes to turn `base`/`themes` into the selected format's content, and `ExportConfiguration` (`format`, `context`, `mimeType`, `data`) to track the current selection. `ExportPalette.tsx` owns the mode-level state (current export selection, copy/download actions); `Export.tsx` (a large, ~1700-line file) owns the format picker and per-option feature flags.

- **Stores**: no dedicated export store — the current export selection is local `ExportPalette` state, recomputed from the live palette props on every format change.
- **Bridges**: export is entirely client-side (no message to the host beyond the generic `POST_MESSAGE` warning toast on a failed clipboard copy); nothing is written back to the document.
- **Analytics**: every format selection is tracked via `trackExportEvent`, keyed off the clicked option's `data-value` (`ExportEvent['feature']`).
- **Platforms**: Figma, Penpot and Sketch get every format. Framer gets a narrower set: CSS, SCSS, Less, Tailwind v3/v4 and JSON tokens (native, DTCG); no Apple, Android or CSV. On every platform, the CSS, SCSS and Less color-space choice includes OKLCH.

## Open questions

*(none)*

## See also

- [Palette](../01-domain-model/palette.md) §8 — the full list of export formats/contexts and their semantic-layer variants
- [Color System](../01-domain-model/color-system.md) — how the semantic layer feeds into these same export formats
- [Inspect](inspect.md) — the sibling read-only mode, reached from the same top-level mode switcher
- [Settings](settings.md) — the palette's color-space setting

## History

| Date | Change |
| --- | --- |
| 2026-08-04 | Created — as-is consolidation from reading `ExportPalette.tsx` and `Export.tsx`, following up on a request to cover the missing Export/Inspect modes |
| 2026-10-28 | Confirmed by the product owner: Framer's export set (SCSS included) and OKLCH in the CSS, SCSS and Less color-space choice |
| 2026-10-29 | Plan, gating and credits content removed; plans live in the docs (free-and-pro) |
| 2026-10-29 | Confirmed by the product owner: the color-space flags switch the color model of the exported code, not plan availability |
| 2026-11-04 | Aligned with the code and the product owner: download file names written down as they are today (naming to improve)
| 2026-11-05 | File naming convention decided by the product owner and applied (POR-687): `<palette>.<role>.<ext>` for every format, `<palette>.zip` for multi-file exports
