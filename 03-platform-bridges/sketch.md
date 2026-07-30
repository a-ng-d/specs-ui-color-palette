# Platform — Sketch

- **Status**: Draft
- **Host repo**: `sketch-ui-color-palette` (`sketch-ui-color-palette.sketchplugin`)

Fill in further with `../TEMPLATE.md` for any development specific to this platform.

## Anchor points already identified

- `config.env.colorMode`: `sketch-light | sketch-dark`
- The host imports `ui-ui-color-palette` as a git submodule
- Platform API: Sketch API (variables, shared styles)

Architecturally, variable and style sync mirror Figma's naming and behavior, but Sketch's underlying capabilities are more limited.

## Syncing to Sketch swatches (variables)

- Each shade of the palette becomes a **document swatch** — Sketch's closest equivalent to a Figma variable.
- **No theme/mode concept**: Sketch has no way for one swatch to hold a different value per theme. Themed shades still get created as swatches, but flat — there's no true parity with Figma's "one variable, several mode values."
- No contrast/lightness documentation attached: Sketch swatches don't support the kind of rich description Figma variables carry.
- Safe to re-run, matched by name.

## Syncing to Sketch shared styles

- Same naming convention and intent as variables, producing **Shared Layer Styles** instead.
- Same limitations: no theme switching, no description.

## Document generation

- Builds the documentation board directly on the current page and centers the view on it.
- Instead of a version checkpoint (Sketch has no version-history equivalent to Figma/Penpot), the whole document is saved.

## Storage

- The palette configuration lives **on the Sketch document** — never tied to the individual user's account.

## See also

- [Bridge catalog](bridge-actions.md) — the shared UI↔host bridges this platform also implements
- [Figma](figma.md), [Penpot](penpot.md) — platforms with true theme/mode support, unlike Sketch's flat swatches/styles
- [Framer](framer.md) — the other platform with no theme/mode concept, but user-scoped storage instead of document-scoped
- [Palette](../01-domain-model/palette.md) — what gets synced (shades, themes) and where it comes from
