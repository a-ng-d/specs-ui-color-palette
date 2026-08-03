# Feature — Image extraction (dominant-color palette seeding)

- **Status**: Draft (as-is behavior, with a confirmed silent-failure gap flagged — see Acceptance criteria)
- **Package(s) concerned**: `ui-ui-color-palette`
- **UI module**: `src/ui/services/ImagePalette.tsx`
- **Related spec**: [Creation](creation.md) (overview, shared cost model), [Colors](colors.md)

## Problem

One of four alternative ways to seed a new palette: provide an image, get its 5 dominant colors as source colors. It works as expected, but pasting a non-PNG image from the clipboard fails silently — no error, no feedback, just nothing happens.

## User flow

1. The user provides an image one of two ways: pasting an image from the clipboard (`Ctrl/Cmd+V`), or via an image already selected on the host canvas (pushed in by the host, no in-app upload button exists).
2. Up to 5 dominant colors are extracted client-side from the image and shown as a preview alongside the source image.
3. The user commits with "Use this palette" — same handoff as the other three methods (see Creation): the palette is created and opened immediately.

## Rules

- There is **no file-picker/upload button** in this UI — the only two ways in are clipboard paste and a host-pushed canvas selection (`GET_IMAGE_HASH` message).
- Clipboard paste only accepts `image/png` — any other image MIME type (JPEG, GIF, WebP, etc.) is silently ignored: the paste handler checks the type and simply does nothing if it doesn't match, with no error or toast shown to the user. *(confirmed gap — see Acceptance criteria)*
- Extraction always returns up to 5 dominant colors (`DominantColors.extract(buffer, 5)`), regardless of source (paste or canvas selection).
- Generated source colors are tagged `source: 'IMAGE'`, with hue/chroma shift both reset to 0 (not locked) — same pattern as Color wheel and AI generation.
- The extraction fee (`imageColorsExtract`) is charged once per successful extraction (paste or canvas selection alike), before the user decides whether to commit; the `paletteCreate` fee is charged separately, only on commit — see Creation's shared cost model.

## Acceptance criteria

- [ ] Given a PNG image pasted from the clipboard, when extraction succeeds, then up to 5 dominant colors are shown with the source image.
- [ ] Given a non-PNG image pasted from the clipboard, when the paste happens, then the user sees a clear message explaining why nothing happened. *(currently silent — confirmed gap, fix pending)*
- [ ] Given an image pushed in from the host canvas selection, when extraction runs, then it behaves identically to a clipboard paste (same 5-color cap, same fee).
- [ ] Given the local-palette quota is reached, when the user tries to commit, then the action is blocked with a trial/upgrade prompt.

## Out of scope

- Adding an in-app file-picker/upload button as a third way to provide an image.
- Supporting clipboard paste for image formats beyond PNG.
- Changing the fixed 5-dominant-color extraction cap.

## Implementation notes

Reuses the engine's `SourceColorConfiguration` shape; extraction is delegated to a `DominantColors.extract` helper operating on the raw image buffer, client-side.

- **Stores**: reads `$palette` only to attach the current in-progress exchange data to the `CREATE_PALETTE` message; writes nothing itself.
- **Bridges**: sends `CREATE_PALETTE` with `{ sourceColors, exchange }` — same shape as the other three creation methods. `GET_IMAGE_HASH` is the inbound message carrying a host-selected canvas image.
- **Analytics**: `EXTRACT_DOMINANT_COLORS` (commit) via `trackImportEvent`; the underlying `CREATE_PALETTE` action itself is tracked separately.
- **Credits**: `imageColorsExtract` (per successful extraction, paste or canvas) + `paletteCreate` (commit) — see Creation's shared cost model.
- **Platforms**: the canvas-image-selection entry point necessarily depends on host support for pushing an image reference — not detailed further here.

## Open questions

*(none)*

## See also

- [Creation](creation.md) — the shared overview, cost model, and the three sibling methods
- [Colors](colors.md) — the analogous default-seed-color and hue/chroma-shift behavior in the main editor

## History

| Date | Change |
| --- | --- |
| 2026-08-04 | Created — as-is consolidation from reading `ImagePalette.tsx`, following up on a request to cover the creation services |
