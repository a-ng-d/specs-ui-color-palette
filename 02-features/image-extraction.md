# Feature — Image extraction (dominant-color palette seeding)

- **Status**: Implemented (as-is behavior, with a confirmed silent-failure gap flagged — see Acceptance criteria)
- **Package(s) concerned**: `ui-ui-color-palette`
- **UI module**: `src/ui/services/ImagePalette.tsx`
- **Related spec**: [Creation](creation.md) (overview), [Colors](colors.md)

## Problem

One of four alternative ways to seed a new palette: provide an image, get its 5 dominant colors as source colors. It works as expected, but pasting a non-PNG image from the clipboard fails silently — no error, no feedback, just nothing happens. Dropping or uploading a non-PNG file shows an error.

## User flow

1. The user provides an image one of three ways: dropping a PNG file on the drop zone or clicking it to browse for one (upload), pasting an image from the clipboard (`Ctrl/Cmd+V`), or via an image already selected on the host canvas (pushed in by the host; Figma, Sketch and Framer).
2. Up to 5 dominant colors are extracted client-side from the image and shown as a preview alongside the source image.
3. The user commits with "New UI Color Palette" (or "Start with this palette" when there is no palette yet) — same handoff as the other three methods (see Creation): the palette is created and opened immediately.

## Rules

- The UI has a **drop zone with a file picker** ("Drag & drop your .png file here", "or click to browse"), one image at a time and PNG only. A JPEG, or any other format, shows a message: a warning ("The file was not imported. Retry dragging & dropping a .png file here.") when the file is not accepted, and an error ("The image could not be loaded. Make sure the image is a .png and drop it here.") when it cannot be loaded. The two other ways in are clipboard paste and a host-pushed canvas selection (`GET_IMAGE_HASH` message).
- Clipboard paste only accepts `image/png` — any other image MIME type (JPEG, GIF, WebP, etc.) is silently ignored, unlike the drop zone, which shows an error: the paste handler checks the type and simply does nothing if it doesn't match, with no error or toast shown to the user. *(confirmed gap — see Acceptance criteria)*
- Extraction always returns up to 5 dominant colors (`DominantColors.extract(buffer, 5)`), regardless of source (paste or canvas selection).
- Generated source colors are tagged `source: 'IMAGE'`, with hue/chroma shift both reset to 0 (not locked) — same pattern as Color wheel and AI generation.

## Acceptance criteria

- [ ] Given a PNG image pasted from the clipboard, when extraction succeeds, then up to 5 dominant colors are shown with the source image.
- [x] Given a JPEG or any other non-PNG file dropped or uploaded through the drop zone, when the file is rejected, then the user sees a message asking for a PNG.
- [ ] Given a non-PNG image pasted from the clipboard, when the paste happens, then the user sees a clear message explaining why nothing happened. *(currently silent — confirmed gap, fix pending)*
- [ ] Given an image pushed in from the host canvas selection, when extraction runs, then it behaves identically to a clipboard paste (same 5-color cap).

## Out of scope

- Supporting clipboard paste for image formats beyond PNG.
- Changing the fixed 5-dominant-color extraction cap.

## Implementation notes

Reuses the engine's `SourceColorConfiguration` shape; extraction is delegated to a `DominantColors.extract` helper operating on the raw image buffer, client-side.

- **Stores**: reads `$palette` only to attach the current in-progress exchange data to the `CREATE_PALETTE` message; writes nothing itself.
- **Bridges**: sends `CREATE_PALETTE` with `{ sourceColors, exchange }` — same shape as the other three creation methods. `GET_IMAGE_HASH` is the inbound message carrying a host-selected canvas image.
- **Analytics**: `EXTRACT_DOMINANT_COLORS` (commit) via `trackImportEvent`; the underlying `CREATE_PALETTE` action itself is tracked separately.
- **Platforms**: the canvas-image-selection entry point necessarily depends on host support for pushing an image reference — not detailed further here.

## Open questions

*(none)*

## See also

- [Creation](creation.md) — the shared overview and the three sibling methods
- [Colors](colors.md) — the analogous default-seed-color and hue/chroma-shift behavior in the main editor

## History

| Date | Change |
| --- | --- |
| 2026-08-04 | Created — as-is consolidation from reading `ImagePalette.tsx`, following up on a request to cover the creation services |
| 2026-10-29 | Plan, gating and credits content removed; plans live in the docs (free-and-pro) |
| 2026-11-04 | Aligned on the code and the product owner: the drop zone and its file picker ship, a non-PNG file gives an error, the create button is "New UI Color Palette" ("Start with this palette" when there is no palette) |
