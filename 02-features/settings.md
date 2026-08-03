# Feature — Settings (palette settings)

- **Status**: Implemented
- **Package(s) concerned**: `ui-ui-color-palette`
- **UI module**: `src/ui/modules/settings`
- **Related spec**: [Palette](../01-domain-model/palette.md) (§2, §3, §5)

## Problem

Palette-level settings (name, description, color space, vision simulation, algorithm version, text colors, deletion) mostly work as expected, but the relationship between "global palette" settings and "active theme" settings is under-communicated to the user, and one validation path (invalid hex input) hasn't been fully verified. Note: "color settings" is a global-settings module (color space/algorithm/vision simulation), not a per-color editor — despite what its name suggests; individual source-color operations live elsewhere, not yet located.

## User flow

1. The user edits the palette's name (64-character limit) and description — each commits on blur/valid input.
2. The user picks a color space (LCH/OKLCH/LAB/OKLAB, HSL/HSV/HSLUV, or CMYK) — a warning appears specifically when HSL is selected.
3. The user picks a vision-simulation mode (none, or one of 8 color-blindness types) — an indicator appears when a non-default theme is active, signaling this setting actually applies to that **active theme**, not the whole palette.
4. The user picks an algorithm version (v1/v2/v3).
5. The user sets light and dark text colors via color pickers — the section title reflects which active theme these apply to.
6. From the danger zone, the user deletes the palette after confirming in a dialog that shows its name.

## Rules

- Name is capped at 64 characters client-side; description has no visible validation.
- Vision-simulation mode and text colors apply to the **currently active theme only**, not the palette as a whole — despite living in a "palette settings" panel.
- Each color-space option and other settings are individually plan-gated.
- Deletion is the only setting behind a confirmation dialog; every other change (including color space and algorithm version) applies immediately.
- The only explicit format validation anywhere in this module is a hex-color regex on the two text-color pickers.
- Dark text's displayed default is `#000` — the 3-digit shorthand form of valid black, not a typo (it's distinct from, but equivalent to, `#000000`).
- The default algorithm version is `v3`, sourced from the app's global config state — not plan-dependent.

## Acceptance criteria

- [ ] Given the name field, when the user types more than 64 characters, then input beyond the limit is rejected.
- [ ] Given the color space dropdown, when HSL is selected, then a warning message is shown.
- [ ] Given a non-default theme is active, when the user opens vision-simulation mode or text colors, then an indicator makes clear the setting applies to that theme, not the whole palette.
- [ ] Given a text-color hex input, when the user enters an invalid hex value, then the update is rejected and the previous valid value is preserved. *(currently unverified whether the invalid value silently gets sent anyway — needs confirming before this criterion can be marked met)*
- [ ] Given the danger zone, when the user clicks delete, then a confirmation dialog appears showing the palette's name (or a translated placeholder if unnamed), and deletion only proceeds after confirming.

## Out of scope

- Documenting individual source-color operations (rename/remove/add/hex/hue/chroma/reorder) — now covered in [Colors](colors.md).
- Adding confirmation dialogs to non-destructive settings changes (color space, algorithm version) — today only deletion has one, and that's not being revisited here.
- Making the global-vs-theme-scoped distinction more visible in the UI — flagged as worth doing, not scoped here.

## Implementation notes

Each settings section receives its relevant slice of palette state (name/description; color space + vision mode + algorithm version; light/dark text theme; palette id + name for deletion) plus a shared update callback. All four sections expose their own plan-gating per sub-setting. Not duplicated further here — see `../01-domain-model/palette.md` §5.

None of the four settings sections talks to a bridge or store directly — every change funnels through a single update callback owned by the settings panel's parent, which:

- Rebuilds the entire settings payload on every change (not just the changed field), applies it to the shared palette state, and — for vision-simulation mode and text colors specifically — applies it to the currently active theme only.
- Sends a single generic "settings updated" message to the host side, and tracks the change.
- On the host side, applying a settings update re-reads the palette, updates the active theme and/or the palette's base fields, refreshes its "last updated" timestamp, and notifies the UI.
- Settings are persisted in two steps: first to the shared in-memory store (for immediate UI feedback), then written through to the actual document (the palette) on the host side — this is a genuine round-trip, not local/install-scoped-only storage.
- Deletion sends a dedicated "delete this palette" message; the host removes it and confirms back to the UI, which then closes the confirmation dialog.
- **Plan gating**: each color-setting option is individually plan-gated, prompting a trial/upgrade flow when blocked — no credit consumption identified in this module.
- **Platforms**: no platform-specific branching found in this module — gating is uniform, driven entirely by the shared plan/feature configuration.

## Open questions

*(none — see resolutions in History)*

## See also

- [Palette](../01-domain-model/palette.md) §2/§3/§5 — source color, theme, and global-setting operations
- [Preview](preview.md) — color space and vision-simulation controls are duplicated in the preview panel
- [Palettes](palettes.md) — the Danger Zone deletion action, local-listing counterpart
- [Colors](colors.md) — individual source-color operations, out of scope here
- [Themes](themes.md) — this module's vision-simulation control duplicates a second, independent one living per-theme in `Themes.tsx`

## History

| Date | Change |
| --- | --- |
| 2026-07-27 | Created — as-is consolidation from an agent's code reading |
| 2026-07-30 | Rewritten at a functional level (behavior/edge cases instead of file/line references), internal links added |
| 2026-08-03 | Reformatted to the Problem/User flow/Rules/Acceptance criteria template |
| 2026-08-04 | Resolved all four open questions: source-color operations live in `Colors.tsx` (candidate for its own spec), default dark text `#000` confirmed as valid shorthand black (not a typo), default algorithm version `v3` sourced from global config, and settings updates confirmed to genuinely round-trip (store, then document) |
| 2026-08-04 | Linked new [Colors](colors.md) and [Themes](themes.md) specs; flagged that this module's vision-simulation control duplicates a second, independent one in `Themes.tsx` |
