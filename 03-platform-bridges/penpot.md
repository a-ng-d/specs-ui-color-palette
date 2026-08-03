# Platform — Penpot

- **Status**: Implemented
- **Host repo**: `penpot-ui-color-palette`

Fill in further with `../TEMPLATE.md` for any development specific to this platform.

## Anchor points already identified

- `config.env.colorMode`: `penpot-light | penpot-dark`
- The host imports `ui-ui-color-palette` as a git submodule
- Platform API: Penpot Plugin API — tokens are native to Penpot (unlike Figma)

## Syncing to Penpot tokens

- Each shade of the palette becomes a **design token**, grouped into one or more token **Sets** — Penpot's native equivalent of Figma's variables, though organized differently.
- **No themes**: a single Set holds every token for the palette.
- **With themes**: one Set per theme, each paired with a matching Penpot **Theme** — the direct equivalent of Figma's "one mode per theme," expressed through Penpot's own token/theme model instead of collections and modes.
- Tokens are self-documenting, same as Figma variables: source color vs. generated shade, lightness percentage, WCAG/APCA contrast score.
- Safe to re-run — existing sets, themes, and tokens are reused, never duplicated.

## Syncing to Penpot styles

- Same intent and naming convention as the token sync, but produces plain **library colors** instead of tokens.
- Lighter than tokens: no contrast/lightness documentation attached.

## Document generation

- Same behavior as Figma: builds a documentation board on the canvas, selects it, zooms to it, and checkpoints a version.

## Storage

- Same model as Figma: the palette configuration lives **on the Penpot document**, not tied to the individual user's account.

## See also

- [Bridge catalog](bridge-actions.md) — the shared UI↔host bridges this platform also implements
- [Figma](figma.md) — closest platform in capability (variables/modes vs. Penpot's native tokens/themes)
- [Sketch](sketch.md), [Framer](framer.md) — platforms with a narrower variable/style/theme model
- [Palette](../01-domain-model/palette.md) — what gets synced (shades, themes) and where it comes from
