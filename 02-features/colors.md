# Feature — Colors (source colors editor)

- **Status**: Draft (as-is behavior, with a confirmed validation gap flagged — see Acceptance criteria)
- **Package(s) concerned**: `ui-ui-color-palette`
- **UI module**: `src/ui/contexts/Colors.tsx`
- **Related spec**: [Palette](../01-domain-model/palette.md) (§2), [Scale](scale.md), [Themes](themes.md), [Settings](settings.md)

## Problem

Source-color management is the module previously flagged in `scale.md`/`preview.md`/`settings.md` as "confirmed to live outside those modules, not yet located" — it lives in `Colors.tsx`. It mostly works as expected, but shares a validation gap with the Themes module: an invalid hex entry doesn't just get rejected — it still fires the update message and the analytics event, carrying stale data. This spec documents current behavior and flags that for fixing.

## User flow

1. The user sees the palette's source colors as a sortable, reorderable list, each row showing an editable name (24-character limit) and, on wide enough screens, an inline hex-color picker.
2. On narrow screens the hex picker moves into a "more parameters" panel per color, opened per row instead of inline.
3. From that same "more parameters" panel, the user can: toggle alpha mode (and, once enabled, set a background color used to preview the composited result), shift hue (-180°..180°) and chroma (0-200%) as per-color offsets from the palette-wide default, edit L/C/H numeric fields directly, and write a free-text description.
4. The user adds a new source color (up to a plan-gated limit, default 5) — it's seeded with a fixed default color (a light cyan, not randomized) and a generic incrementing name.
5. The user removes a color from its row, or reorders colors by drag-and-drop.
6. An empty state invites adding the first color when the list is empty.

## Rules

- Name is capped at 24 characters (narrower than the 64-character palette name in Settings); a duplicate name gets " 2" appended once — a second duplicate wouldn't be renamed further (no incrementing counter).
- Hue and chroma shifts here are **per-color offsets** from the palette-wide hue/chroma shift (`props.shift`) — distinct from the palette-wide sliders documented in Scale, which apply globally to every color at once. A shift is flagged "locked" once it diverges from the palette-wide value; Reset clears it back to that shared value.
- Adding a color always seeds the same fixed default color and doesn't touch any "active" state — unlike adding a theme (see Themes), where the new item becomes the sole active one.
- The only format validation in this module is the same hex-color regex used elsewhere (3- or 6-digit hex, `#` optional on input).
- On an invalid hex entry, the palette store is correctly left untouched — but the update message is still sent to the host, and the analytics event still fires, carrying whatever data was captured by the last valid update rather than being suppressed. *(confirmed by reading `updateHexCode`; the identical pattern is present in Themes' background/text-color pickers — see Acceptance criteria)*
- Each of the fields/actions in this module (add/name/hex+LCH/hue-shift/chroma-shift/description/alpha/background/reorder) is individually plan-gated.

## Acceptance criteria

- [ ] Given the source-color list, when the user reorders colors, then the new order is sent and tracked as `REORDER_COLOR`.
- [ ] Given a color's hex input, when the user enters a valid 3- or 6-digit hex, then the color updates and the change is tracked as `UPDATE_HEX`.
- [ ] Given a color's hex input, when the user enters an invalid value, then no update message is sent and no analytics event fires. *(currently a message is sent and an event tracked regardless, using stale data — confirmed gap, fix pending)*
- [ ] Given the source-color quota is reached, when the user tries to add a color, then the action is blocked with a trial/upgrade prompt.
- [ ] Given the L/C/H numeric fields, when any of the three (lightness, chroma, hue) is edited, then the analytics event distinguishes which one changed. *(currently all three are tracked under the same `UPDATE_LCH` feature name — confirmed gap)*

## Out of scope

- Randomizing or making configurable the default seed color used when adding a source color.
- Adding a multi-duplicate-safe renaming scheme (today's " 2" suffix only handles a single collision).
- Any change to how per-color hue/chroma shifts interact with the palette-wide shift sliders in Scale — that relationship is documented, not being redesigned here.

## Implementation notes

Reuses the engine's `ColorConfiguration` and `ShiftConfiguration`. Unlike most other feature modules, this one lives directly under `src/ui/contexts/` as a single file, rather than as a folder under `src/ui/modules/`.

- **Stores**: writes directly to the shared palette store (`colors` key) on every change; no dedicated local component state beyond what each editing control needs.
- **Bridges**: every change funnels through a single `UPDATE_COLORS` message carrying the full colors array (not a diff), followed by a store write and an analytics call — mirrors the Scale/Themes pattern of full-payload messages.
- **Analytics**: all source-color actions are tracked (`ADD_COLOR`, `RENAME_COLOR`, `UPDATE_HEX`, `UPDATE_LCH` ×3, `SHIFT_HUE`, `SHIFT_CHROMA`, `RESET_HUE`, `RESET_CHROMA`, `DESCRIBE_COLOR`, `SWITCH_ALPHA_MODE`, `UPDATE_BACKGROUND_COLOR`, `REMOVE_COLOR`, `REORDER_COLOR`).
- **Platforms**: no platform-specific branching found in this module.

## Open questions

*(none)*

## See also

- [Palette](../01-domain-model/palette.md) §2 — the full list of `SourceColorEvent` actions and the associated bridge
- [Scale](scale.md) — the palette-wide hue/chroma shift sliders that this module's per-color shifts offset from
- [Themes](themes.md) — the sibling module sharing the same file location and the same hex-validation gap
- [Settings](settings.md) — the analogous hex-validation gap noted for text-color pickers, now confirmed here

## History

| Date | Change |
| --- | --- |
| 2026-08-04 | Created — as-is consolidation from reading `Colors.tsx`, following up on the "not yet located" note in scale.md/preview.md/settings.md |
