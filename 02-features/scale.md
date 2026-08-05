# Feature — Scale (scales & presets)

- **Status**: Implemented (as-is behavior, with confirmed dead code and tracking gaps flagged — see below)
- **Package(s) concerned**: `ui-ui-color-palette`
- **UI module**: `src/ui/modules/scale`
- **Related spec**: [Palette](../01-domain-model/palette.md) (§4)

## Problem

Scale editing has real internal inconsistencies that were never resolved: two custom-stop caps where one is now dead code, two different "reset" behaviors depending on which editing view triggered them, and several actions that silently aren't tracked in analytics. This spec documents current behavior and marks what's confirmed to need cleanup.

## User flow

1. The user picks a preset from 18 built-in options (or a fully custom scale) — the scale recomputes from that preset's stops/bounds/easing.
2. The user edits the scale in one of two mutually exclusive views, toggled by a single switch, both operating on the exact same underlying scale:
   - **Raw editing** — direct stop/lightness/hue/chroma editing.
   - **Contrast-ratio editing** — set a target WCAG contrast ratio per stop instead, and the underlying lightness is solved for it.
3. With a custom preset active, the user can add/remove stops (up to a 6-stop plan limit today).
4. The user can reverse all stop values around the scale's midpoint, or reset the scale back to the active preset's defaults.
5. Two independent sliders (hue, chroma) apply a global offset to the whole scale.
6. A separate easing control (curve × velocity), outside both editing views, shapes the distribution of the scale.
7. A keyboard-shortcuts panel is available as a reference — it's purely informational, listing interactions implemented by the underlying slider components.

## Rules

- 6 of the 18 presets show a "(family)" suffix in their name — Material, Material 3 (family "Google"), ADS, ADS Neutral (family "Atlassian"), and Spectrum, Spectrum Neutral (family "Adobe"). Each is set via a per-preset `updateName` flag hardcoded in the switch-preset handler; the other 12 presets (Tailwind, Ant, Bootstrap, Radix, Untitled UI, Open Color, Carbon, Base, Polaris, Fluent, plus the 3 custom starting points) never get the suffix. No single systematic rule ties the flag to the preset's name/family shape — it reads as a manual, one-off choice per preset.
- Custom stops: a hardcoded ceiling of 24 stops exists, but a separate plan-based limit resolves to 6 stops for every plan today — since 6 is reached first, the 24 cap is effectively dead code under the current configuration. This ceiling is a technical limit tied to how much data each host platform can currently persist per palette; it's expected to be revisited if platforms allow storing more data.
- Adding a stop derives its new position from the existing stops; confirmed to stay correct after repeated add/remove cycles on unevenly spaced stops.
- **Reset** behaves differently depending on which view triggered it: raw-editing reset restores stops, chroma, and hue to the preset's defaults; contrast-ratio reset only restores chroma. Neither reapplies the preset's original easing curve — both silently fall back to linear.
- Hue and chroma shifts are **global offsets applied to the whole scale**, not per-stop.
- In contrast-ratio mode, editing a ratio caps how far a single edit can move a stop's lightness relative to its previous value (a minimum jump of 5%, or 10% of the previous lightness, whichever is larger), to avoid a jarring jump from one ratio edit.
- If an edit breaks the stops' monotonic order, the scale is partially re-sorted — mostly interpolated back into order, but still favoring the value the user just typed.

## Acceptance criteria

- [ ] Given a custom preset with fewer than 6 stops, when the user adds a stop, then the addition succeeds up to 6 stops, then blocks with a plan/upgrade prompt.
- [ ] Given the scale is in raw-editing view, when the user resets, then stops, chroma, and hue all return to the active preset's defaults.
- [ ] Given the scale is in contrast-ratio view, when the user resets, then behavior matches raw-editing reset (stops, chroma, and hue all return to defaults). *(currently only chroma resets in this view — confirmed divergence, fix pending)*
- [ ] Given any reset, when it completes, then the preset's original easing curve is reapplied rather than silently falling back to linear. *(currently not the case — confirmed gap, fix pending)*
- [ ] Given the user reverses the stops, when the action is tracked, then the analytics event correctly identifies "reverse" as the triggering action. *(currently the message never records which action triggered it — confirmed gap)*
- [ ] Given the user shifts hue or chroma, when the action completes, then it is tracked in analytics the same way preset/stop/reset changes are. *(currently not tracked at all — confirmed gap)*
- [ ] Given the keyboard-shortcuts panel is open, when it re-renders without the user reopening it, then no additional "opened" analytics event fires. *(currently fires on every re-render — confirmed gap)*
- [ ] Given an edit breaks stop monotonicity, when the scale re-sorts, then stops remain lighter/darker than their neighbors in order, while still favoring the just-edited value.

## Out of scope

- Removing the dead 24-stop cap — kept deliberately as a technical ceiling that may be raised if host platforms allow persisting more data per palette; not cleanup debt.
- Making the "(family)" suffix rule systematic — it stays a manual, per-preset choice (see Rules for the current 6).
- Any change to which runtime applies scale updates on the host side (the embedded `engine-ui-color-palette`), or how it maps to each platform's sandbox.

## Implementation notes

Reuses the engine's `PresetConfiguration`, `ScaleConfiguration` (stop → lightness map), `ShiftConfiguration` (chroma/hue offsets), and `EasingConfiguration`. See `../01-domain-model/palette.md` §4 for the full set of scale-related actions.

- **Stores**: scale edits are written directly to the shared palette store from multiple editing components at once — there's no single dedicated "scale" store, and several components independently subscribe to the same palette state while editing.
- **Bridge**: all scale edits funnel through a single `UPDATE_SCALE` message. On the receiving end, adding/removing a stop also recomputes the scale for themes that aren't currently active, using logic that isn't identical to how the sending side recomputes it — worth double-checking if stop count ever becomes theme-specific.
- **Engine**: the runtime that actually applies scale updates (recomputing stops/lightness/easing) is `engine-ui-color-palette` — it's embedded locally within each platform's plugin (Figma, Penpot, Sketch, Framer each bundle their own copy), not a remote/shared service.
- **Analytics**: preset changes, add/remove stop, and reset are tracked (with the gaps noted above for reverse/hue/chroma/keyboard shortcuts).
- **Platforms/plan**: every control is plan-gated identically across platforms; the actual availability differences live in the shared feature-flag configuration, not in this module.

## Open questions

*(none — see resolutions in History)*

## See also

- [Palette](../01-domain-model/palette.md) §4 — the full list of scale operations tracked as `ScaleEvent`
- [Preview](preview.md) — the contrast-ratio editing mode uses the same WCAG math as the contrast report
- [Bridge messages & analytics events](../04-contracts/events-messages.md) — `ScaleMessage` / `ScaleEvent` payload shape
- [Colors](colors.md), [Themes](themes.md) — the sibling source-color and theme editors, same full-payload bridge pattern
- [Hue/Chroma Distribution](hue-chroma-distribution.md) — Draft spec adding a `LINEAR`/`HYPERBOLA`/`FREE` curve on top of this module's global hue/chroma shift sliders

## History

| Date | Change |
| --- | --- |
| 2026-07-27 | Created — as-is consolidation from an agent's code reading |
| 2026-07-30 | Rewritten at a functional level (behavior/edge cases instead of file/line references), internal links added |
| 2026-08-03 | Reformatted to the Problem/User flow/Rules/Acceptance criteria template |
| 2026-08-04 | Resolved all four open questions: 24-stop cap is deliberate (tied to platform storage limits), the 6 family-suffixed presets identified (Material, Material 3, ADS, ADS Neutral, Spectrum, Spectrum Neutral), the applying runtime is the embedded `engine-ui-color-palette`, and the stop-adding formula confirmed correct after repeated add/remove cycles |
| 2026-08-04 | Linked new [Colors](colors.md) and [Themes](themes.md) specs |
