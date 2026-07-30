# Feature — Color System (semantic layer)

- **Status**: Draft — engine/API/MCP available today, no design-tool UI yet (this spec doubles as the forward-looking scope for that UI)
- **Package(s) concerned**: `engine-ui-color-palette` (resolution logic), `api-ui-color-palette` / `mcp-ui-color-palette` (exposure), `claude-ui-color-palette` (guided workflow), `ui-ui-color-palette` (not implemented yet)
- **Related spec**: [Color System (domain model)](../01-domain-model/color-system.md), [Platform bridges](../03-platform-bridges/bridge-actions.md) (how primitives sync today — the semantic layer doesn't sync anywhere yet)

## 1. Context

A palette (`PaletteData`) only has primitives: colors and shades, identified by position (`blue:500`). Teams don't build products against `blue:500` — they build against meaning: "the default brand color," "a destructive action," "a disabled surface." The Color System is the naming layer that sits on top of the palette and gives shades that meaning, so downstream code/design-tool exports can carry semantic names instead of raw shade references.

It's additive and non-destructive: building or changing a Color System never touches the underlying palette.

## 2. Current behavior (as-is)

### What exists today

- A **taxonomy**: one or more named dimensions (e.g. Role, Prominence, State, Surface/Content), each with a fixed set of named members (e.g. Role → Brand, Danger, Success). Combining one member from each dimension produces every possible semantic token path (e.g. `role=brand` × `state=hover` → the "brand, hover" token).
- A **binding** per meaningful path: which primitive shade backs that token (`colorId:shadeName`, e.g. `blue:500`), optionally a different shade per theme (light theme → `blue:500`, dark theme → `blue:400`), an optional human description, and an optional "excluded from export" flag.
- **Resolution**: every combination of taxonomy members is expanded into a token; each token is resolved against every theme in the palette. A path without a binding, or a binding pointing at a shade that doesn't actually exist in a given theme, resolves to "no value" for that theme rather than failing the whole system.
- Exposed today only through the API/MCP tool `get_color_system` and the Claude orchestrator's guided flow (taxonomy pattern selection → binding proposal → refinement) — no in-app (Figma/Penpot/Sketch/Framer) UI to define or edit it yet.
- Once resolved, the Color System can feed code/token export (`generate_code`) to produce a primitives file **and** a semantic file per format, alongside the primitives-only export that already exists.

### Edge cases already handled

- Taxonomy with zero groups → no tokens at all (not an error, just an empty system).
- A token with no matching binding → still appears in the output, with a null value per theme, instead of being silently dropped — so a taxonomy can be defined before every path is bound, and gaps stay visible.
- A binding whose shade reference doesn't exist for a given theme (e.g. a theme that dropped a color the others have) → that theme resolves to null for that token, other themes are unaffected.
- `isExcluded` tokens are still fully resolved internally (so their value is known) — only export is expected to skip them. Whether "excluded" should still get its per-theme overrides resolved, or short-circuit entirely, isn't settled (see Open questions).

## 3. Proposal (to-be) — design-tool UI

Not built yet. Scope to settle before implementation:

- An in-app equivalent of the orchestrator's guided flow: pick or define a taxonomy, get binding suggestions from the existing palette, refine, confirm.
- A way to view the resulting token matrix (path × theme → shade) directly in the plugin, mirroring what the orchestrator already renders as a text matrix.
- Sync of the semantic layer into each platform's native mechanism — this doesn't exist today even at the bridge level (see `../03-platform-bridges/`): Figma/Penpot could alias semantic tokens to primitive variables/tokens, Sketch/Framer have no aliasing concept and would need a flat naming convention instead, similar to how their primitive sync already degrades relative to Figma/Penpot (theme/mode support).

## 4. Data model

See [`../01-domain-model/color-system.md`](../01-domain-model/color-system.md) for the full shape (`SystemConfiguration`, `SystemData`, taxonomy schema, binding format). Not duplicated here.

## 5. Impact

- **Bridges**: none yet for the semantic layer specifically — primitive sync bridges are unaffected until platform sync is scoped (see Proposal above).
- **API / MCP**: already covered by `get_color_system`, and by `generate_code` when a system is passed alongside `base`/`themes`.
- **Platforms**: not yet available anywhere as a design-tool artifact — only as generated code/tokens.
- **Credits / plan**: not documented yet — to confirm whether resolving/exporting a Color System consumes credits the same way palette generation does.

## 6. Open questions

- Does an `isExcluded` token still get its per-theme overrides resolved, or should resolution short-circuit for it?
- What should happen in-app when a taxonomy references a palette that later loses a color or a shade a binding points to — surfaced as a warning, silently nulled (current API behavior), or blocking?
- Should the design-tool UI allow editing bindings by hand, or stay guided-only (propose → refine → confirm) like the orchestrator?
- How does semantic sync map onto each platform's native capabilities (aliasing on Figma/Penpot vs. flat naming on Sketch/Framer) — same question the primitive sync already answers, not yet answered for semantics.

## See also

- [Color System (domain model)](../01-domain-model/color-system.md) — taxonomy/binding shape, `get_color_system` contract
- [Palette](../01-domain-model/palette.md) — the primitives this layer resolves against
- [Platform bridges](../03-platform-bridges/bridge-actions.md) — where semantic sync would eventually plug in, alongside the existing primitive sync
- [MCP tools](../04-contracts/mcp-tools.md) — `get_color_system`, and `generate_code` with a system attached

## 7. History

| Date | Change |
| ---- | ------ |
| 2026-07-30 | Created — functional spec covering the existing engine/API/MCP behavior and scoping the not-yet-built design-tool UI |
| 2026-07-30 | Internal links added |
