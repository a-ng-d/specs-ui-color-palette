# Feature — Settings (palette settings)

- **Status**: Draft (as-is consolidation — no behavior change proposed at this stage)
- **Package(s) concerned**: `ui-ui-color-palette`
- **UI module**: `src/ui/modules/settings`
- **Related spec**: [Palette](../01-domain-model/palette.md) (§2, §3, §5)

## ⚠️ Correction vs. the initial stub

The initial stub assumed, from the module's naming alone, that "color settings" handles individual source-color operations (rename/remove/add/hex/hue/chroma/reorder). **This isn't the case.** That section actually covers three *global* palette settings: color space, vision-simulation mode, and algorithm version. Individual source-color operations are wired elsewhere, outside this module (not yet located — see §6). Confirmed by the author as intended: this section is deliberately scoped to global color settings, not per-color editing. Not a bug — just a name that reads as broader than it is.

## 1. Context

This spec documents palette-level settings: name/description, color space, color-blindness simulation, algorithm version, light/dark text colors, and palette deletion. Consolidation only, no proposed evolution.

## 2. Current behavior (as-is)

### Name & description

Two fields — name (64-character limit) and a free-form multi-line description — each committed on blur/valid-input, independently plan-gated, no visible debounce.

### Global color settings

- **Color space**: a dropdown across three families (LCH/OKLCH/LAB/OKLAB, HSL/HSV/HSLUV, CMYK), with a warning shown specifically when HSL is selected. Each option is individually plan-gated.
- **Vision-simulation mode**: none, or one of 8 color-blindness simulation types. An info indicator appears when a non-default theme is active, signaling that this setting is actually carried by the **active theme**, not by the palette as a whole.
- **Algorithm version**: a version picker (v1/v2/v3) for the palette-generation algorithm.

### Contrast (text colors)

Two color pickers — light text and dark text — each committed on pick/blur/valid-input, with an explanatory note. The section title reflects which active theme these colors apply to, reinforcing the same "this is theme-scoped, not palette-wide" pattern as vision simulation.

### Danger zone

A single destructive action: deleting the palette, behind a confirmation dialog that shows the palette's name (with a translated fallback if unnamed). No other destructive action exists here (no full reset, no bulk theme purge).

## 3. Proposal (to-be)

None — consolidation spec.

## 4. Data model

Each settings section receives its relevant slice of palette state (name/description; color space + vision mode + algorithm version; light/dark text theme; palette id + name for deletion) plus a shared update callback. All four sections expose their own plan-gating per sub-setting. Not duplicated further here — see `../01-domain-model/palette.md` §5.

## 5. Impact

None of the four settings sections talks to a bridge or store directly — every change funnels through a single update callback owned by the settings panel's parent, which:

- Rebuilds the entire settings payload on every change (not just the changed field), applies it to the shared palette state, and — for vision-simulation mode and text colors specifically — applies it to the **currently active theme only**, not the palette as a whole.
- Sends a single generic "settings updated" message to the host side, and tracks the change.
- On the host side, applying a settings update re-reads the palette, updates the active theme and/or the palette's base fields, refreshes its "last updated" timestamp, and notifies the UI. Color and theme operations elsewhere follow the same full-replacement pattern.
- Deletion sends a dedicated "delete this palette" message; the host removes it and confirms back to the UI, which then closes the confirmation dialog.
- **Validation**: the only explicit format check anywhere in this module is a hex-color regex on the two text-color pickers — behavior when that check fails (whether the invalid value still gets sent) isn't fully verified yet (see §6). Name is capped at 64 characters client-side; description has no visible validation.
- **Plan gating**: each color-setting option is individually plan-gated, prompting a trial/upgrade flow when blocked — no credit consumption identified in this module.
- **Platforms**: no platform-specific branching found in this module — gating is uniform, driven entirely by the shared plan/feature configuration.

## 6. Open questions

- ~~Where do individual source-color operations (rename/remove/add/hex/hue/chroma/reorder) actually live, since they're not in this module despite the initial assumption~~ — still to locate; may warrant its own "colors" spec once found.
- The displayed default for dark text uses a placeholder hex value that's very close to, but not exactly, valid black — likely a typo with limited real impact (a real value is almost always set) — worth a quick fix regardless.
- When the text-color hex regex fails validation, does the update still get applied/sent with the old value mutated in place, or is it fully blocked? Needs verifying at runtime before relying on it.
- Only palette deletion has a confirmation dialog — other arguably risky changes (color space, algorithm version) apply immediately. Intended?
- The relationship between "global" palette settings and "active theme" settings (vision simulation and text colors silently apply to whichever theme is currently active) is only signaled via a small inline hint — worth making explicit, since it's easy to miss that you're editing the active theme rather than the whole palette.
- Where does the default algorithm version actually come from — global config, or plan-dependent?
- Settings updates appear to go through local, install-scoped storage rather than a live round-trip to the actual document/canvas in the current read of this code — worth confirming whether that's dev/test-only behavior or genuinely how production behaves today, and whether it's consistent across Figma/Penpot/Sketch/Framer.

## See also

- [Palette](../01-domain-model/palette.md) §2/§3/§5 — source color, theme, and global-setting operations
- [Preview](preview.md) — color space and vision-simulation controls are duplicated in the preview panel
- [Palettes](palettes.md) — the Danger Zone deletion action, local-listing counterpart

## 7. History

| Date | Change |
| --- | --- |
| 2026-07-27 | Created — as-is consolidation from an agent's code reading |
| 2026-07-30 | Rewritten at a functional level (behavior/edge cases instead of file/line references), internal links added |
