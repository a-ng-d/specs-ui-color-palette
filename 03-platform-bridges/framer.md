# Platform — Framer

- **Status**: Implemented
- **Host repo**: `framer-ui-color-palette`

Fill in further with `../TEMPLATE.md` for any development specific to this platform.

## Anchor points already identified

- `config.env.colorMode`: `framer-light | framer-dark`
- The host imports `ui-ui-color-palette` as a git submodule
- Platform API: Framer Plugin API (color styles) — no notion of native variables/tokens like Figma or Penpot, and only produces styles (no document-side theme/mode API)

## Syncing to Framer color styles

- Framer only supports **Color Styles** — no variables, no tokens, no multi-theme model.
- **Light/dark inversion**: a Framer color style only ever holds two values, light and dark. To still represent a full theme ramp, each style pairs the shade's own color as the "light" value with the *mirror* shade from the same ramp (e.g. the darkest shade for the lightest, and vice versa) as the "dark" value. Switching Framer's light/dark mode then approximates switching between the palette's themes, even though Framer has no native concept of more than two.
  - The source shade has no mirror, so its light and dark values are identical.
- If the user hasn't granted Framer's color-style permission, no styles are created.
- Same naming convention as the other platforms. Safe to re-run.

## Document generation

- Same documentation-board behavior as the other platforms, but requires broader canvas permissions (frames, text, images, custom data). If any permission is missing, generation is blocked outright rather than degrading.
- No version checkpoint — Framer has no version-history equivalent here.

## Storage

- **Key divergence from every other platform**: the palette configuration is stored **for the user** (browser-level storage), not on the Framer document — Framer's per-document storage is too size-constrained to hold a full palette.
- Practical consequence: switching browser or device, or clearing site data, can lose access to a palette created in Framer — unlike Figma, Penpot, and Sketch, where the palette travels with the file itself.

## See also

- [Bridge catalog](bridge-actions.md) — the shared UI↔host bridges this platform also implements
- [Figma](figma.md), [Penpot](penpot.md) — platforms with true theme/mode support and document-scoped storage
- [Sketch](sketch.md) — the other platform with no theme/mode concept, but document-scoped storage unlike Framer
- [Palette](../01-domain-model/palette.md) — what gets synced (shades, themes) and where it comes from
