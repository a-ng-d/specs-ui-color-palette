# Feature — Scale (scales & presets)

- **Status**: Draft (as-is consolidation — no behavior change proposed at this stage)
- **Package(s) concerned**: `ui-ui-color-palette`
- **UI module**: `src/ui/modules/scale`
- **Related spec**: [Palette](../01-domain-model/palette.md) (§4)

## 1. Context

This spec documents how scale editing currently works: preset selection, editing lightness/hue/chroma, the "contrast ratio" alternative mode, easing, and custom stops. It's the module with the most unresolved internal inconsistencies found during research (see §6) — settle those before building on top of it.

## 2. Current behavior (as-is)

### Two ways to edit the same scale

Editing a theme's scale has **two mutually exclusive views**, switched by a single toggle:
- **Raw editing** — preset picker, direct stop/lightness/hue/chroma editing.
- **Contrast-ratio editing** — the same stops, but driven indirectly: you set a target WCAG contrast ratio per stop and the underlying lightness is solved for it. No presets, no hue/chroma in this view.

Both views edit the exact same underlying scale — switching between them doesn't create two separate states.

### Presets

- 18 built-in presets (Material, Material 3, Tailwind, Ant, Bootstrap, Radix, Untitled UI, Open Color, ADS/ADS Neutral, Spectrum/Spectrum Neutral, Carbon, Base, Polaris, Fluent, plus 3 "Custom" variants) plus fully custom stops.
- 6 of the 18 presets show a "(family)" suffix in their name, the other 12 don't — the exact rule distinguishing them isn't documented (see §6).
- Selecting a preset recomputes the scale from that preset's stops/bounds/easing.

### Custom stops

- Only editable when a custom preset is active.
- Two caps currently coexist: a hardcoded ceiling of 24 stops, and a separate plan-based limit that currently resolves to 6 stops for every plan. Since 6 is reached first, **the 24 cap is effectively unreachable today** — confirmed dead under the current configuration, status as deliberate future-proofing vs. cleanup candidate is still open.
- Adding a stop derives its position from the existing stops; whether this stays correct after repeated add/remove cycles on unevenly spaced stops hasn't been verified (see §6).

### Reverse & reset

- **Reverse** mirrors every stop's value around the scale's midpoint.
- **Reset** behaves differently depending on which view triggered it: the raw-editing reset restores stops, chroma, and hue to the preset's defaults; the contrast-ratio reset only restores chroma. Whether this divergence is intentional is still open (see §6).
- Neither reset re-applies the preset's original easing curve — both silently fall back to linear.

### Hue & chroma shift

- Two independent sliders apply a **global offset** to the whole scale (not per-stop): hue in ±180°, chroma 0–200% (default 100%).

### Contrast-ratio mode

- Precomputes, for every stop, the WCAG ratio against the theme's light and dark text colors.
- Editing a ratio solves back for the lightness that produces it, capping how far a single edit can move a stop relative to its previous value (a minimum jump of 5%, or 10% of the previous lightness, whichever is larger) — this keeps one big ratio edit from producing a jarring lightness jump.
- If an edit breaks the stops' monotonic order (each stop should stay lighter/darker than its neighbors), the scale is partially re-sorted: mostly interpolated back into order, but still favoring the value the user just typed.

### Easing

- A separate curve (Linear / Ease-in / Ease-out / Ease-in-out) × velocity (Sine/Quad/Cubic) control, applied on top of the scale, lives outside the raw/contrast-ratio views (in the shared scale panel, alongside the raw/contrast-ratio switch itself).

### Keyboard shortcuts panel

- Purely informational — it lists available interactions (drag a stop, select, deselect, tab between stops, type a value, nudge by 1) but doesn't implement any shortcut logic itself; that behavior lives in the underlying slider components.

## 3. Proposal (to-be)

None — consolidation spec.

## 4. Data model

Reuses the engine's `PresetConfiguration`, `ScaleConfiguration` (stop → lightness map), `ShiftConfiguration` (chroma/hue offsets), and `EasingConfiguration`. See `../01-domain-model/palette.md` §4 for the full set of scale-related actions.

## 5. Impact

- **Stores**: scale edits are written directly to the shared palette store from multiple editing components at once — there's no single dedicated "scale" store, and several components independently subscribe to the same palette state while editing.
- **Bridge**: all scale edits funnel through a single `UPDATE_SCALE` message. On the receiving end, adding/removing a stop also recomputes the scale for themes that aren't currently active, using logic that isn't identical to how the sending side recomputes it — worth double-checking if stop count ever becomes theme-specific.
- **Analytics**: preset changes, add/remove stop, and reset are tracked. Reverse is tracked but the underlying message never says which action triggered it. Hue/chroma shifts and the easing action are **not tracked** at all today.
- **Platforms/plan**: every control is plan-gated identically across platforms; the actual availability differences live in the shared feature-flag configuration, not in this module.

## 6. Open questions

- Raw-editing reset touches stops+chroma+hue, contrast-ratio reset only touches chroma — intended divergence or tech debt?
- Neither reset reapplies the preset's easing — intentional fallback to linear, or an oversight?
- Reverse-stops is tracked without recording which action caused it — worth fixing if that analytics gap matters.
- Hue/chroma shifts and the easing action aren't tracked — deliberate exclusion from "Scale Updated" analytics, or a gap?
- The keyboard-shortcuts panel tracks its "opened" analytics event on every re-render while open, not just once on open — likely inflates the count; worth confirming before trusting that metric.
- ~~Two custom-stop caps (24 vs. plan-based)~~ — **Resolved**: the plan-based cap currently resolves to 6 for every plan, making the 24 cap unreachable today. Still open whether the 24 cap is deliberate future-proofing or dead code to remove.
- Does the stop-adding formula stay correct after repeated add/remove cycles on unevenly spaced stops?
- What's the exact rule distinguishing the 6 "family"-suffixed presets from the other 12?
- Which runtime actually applies scale updates on the host side, and how does it relate to the per-platform sandboxes (Figma/Penpot/Sketch/Framer)?

## See also

- [Palette](../01-domain-model/palette.md) §4 — the full list of scale operations tracked as `ScaleEvent`
- [Preview](preview.md) — the contrast-ratio editing mode uses the same WCAG math as the contrast report
- [Bridge messages & analytics events](../04-contracts/events-messages.md) — `ScaleMessage` / `ScaleEvent` payload shape

## 7. History

| Date | Change |
| --- | --- |
| 2026-07-27 | Created — as-is consolidation from an agent's code reading |
| 2026-07-30 | Rewritten at a functional level (behavior/edge cases instead of file/line references), internal links added |
