# Platform — Figma / FigJam

- **Status**: Implemented
- **Host repo**: `figma-ui-color-palette`

Fill in further with `../TEMPLATE.md` for any development specific to this platform (anything that only exists here, or diverges from the common behavior described in `02-features/`).

## Anchor points already identified

- Possible `Editor` values for this platform: `figma | figjam | dev | dev_vscode | buzz`
- `config.env.colorMode`: `figma-light | figma-dark`
- The host imports `ui-ui-color-palette` as a git submodule
- Platform API: Figma Plugin API (variables, styles)

## Syncing to Figma variables

- Each shade of the palette becomes a **Figma Variable**, grouped in one Variable Collection named after the palette.
- If the palette has themes, each theme becomes a **mode** on that same collection — one variable, several mode values — instead of duplicating variables per theme.
- Every variable is self-documenting: its description states whether it's the source color or a generated shade, and for generated shades, the lightness percentage used and its WCAG/APCA contrast score. Anyone inspecting the variable in Figma sees this without reopening the plugin.
- **Edge case**: Figma caps how many modes a collection can hold depending on the workspace's plan. If a theme can't get its own mode, that theme is skipped with a warning rather than failing the whole sync.
- Re-running the sync is safe — existing variables and modes are reused, never duplicated.

## Syncing to Figma styles

- Same idea as variables, but produces **Paint Styles** instead — used when the palette has no themes, or as a flatter per-theme alternative when it does (one style per theme × color × shade, rather than one variable holding several mode values).
- Same self-documenting description as variables.
- Also safe to re-run.

## Document generation

- Produces a visual documentation board of the palette directly on the canvas, then selects it and zooms the viewport to it.
- Each generation is checkpointed in Figma's version history, so refreshing the documentation doesn't silently overwrite the previous version.

## Storage

- The palette configuration lives **on the Figma document**, shared with everyone who has access to the file — not tied to the individual user's account.

## See also

- [Bridge catalog](bridge-actions.md) — the shared UI↔host bridges this platform also implements
- [Penpot](penpot.md) — closest platform in capability (native token/theme system vs. Figma's collection/mode system)
- [Sketch](sketch.md), [Framer](framer.md) — platforms with a narrower variable/style/theme model than Figma's
- [Palette](../01-domain-model/palette.md) — what gets synced (shades, themes) and where it comes from
