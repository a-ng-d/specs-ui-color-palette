# Feature — Color System (semantic layer)

- **Status**: Draft — engine/API/MCP available today, no design-tool UI yet (this spec doubles as the forward-looking scope for that UI)
- **Package(s) concerned**: `engine-ui-color-palette` (resolution logic), `api-ui-color-palette` / `mcp-ui-color-palette` (exposure), `claude-ui-color-palette` (guided workflow), `ui-ui-color-palette` (not implemented yet)
- **Related spec**: [Color System (domain model)](../01-domain-model/color-system.md), [Platform bridges](../03-platform-bridges/bridge-actions.md) (how primitives sync today — the semantic layer doesn't sync anywhere yet)

## Problem

A palette only has primitives: colors and shades, identified by position (`blue:500`). Teams don't build products against `blue:500` — they build against meaning: "the default brand color," "a destructive action," "a disabled surface." The engine, API, and MCP layer can already resolve that meaning (a Color System), but there's no way to define or edit one from inside a design tool — only through the API/MCP or the Claude orchestrator's guided flow.

## User flow

### What already works (engine/API/MCP)

1. A taxonomy is defined: one or more named dimensions (e.g. Role, Prominence, State), each with a fixed set of named members (e.g. Role → Brand, Danger, Success).
2. Each meaningful combination of members (a "path") is bound to a primitive shade — optionally a different shade per theme, an optional description, an optional "excluded from export" flag.
3. The system resolves every path against every theme, producing a full token matrix.
4. The resolved system can feed code/token export, producing a primitives file **and** a semantic file per format.

Today this only happens through the API/MCP tool `get_color_system` or the Claude orchestrator's guided flow (pick a taxonomy pattern → get binding suggestions → refine → confirm) — never inside Figma/Penpot/Sketch/Framer directly.

### What's proposed (design-tool UI)

1. In-app, the user picks or defines a taxonomy, gets binding suggestions from the current palette, refines them, and confirms — mirroring the orchestrator's guided flow.
2. The user can view the resulting token matrix (path × theme → shade) directly in the plugin.
3. The user syncs the semantic layer into the platform's native mechanism, same as primitives already sync today.

## Rules

- The Color System is additive and non-destructive: building or changing it never touches the underlying palette.
- A taxonomy with zero groups produces zero tokens — not an error, just an empty system.
- A token with no matching binding still appears in the output, with a null value per theme, instead of being silently dropped — so a taxonomy can be defined before every path is bound, and gaps stay visible.
- A binding whose shade reference doesn't exist for a given theme (e.g. a theme that dropped a color the others have) resolves to null for that token in that theme only; other themes are unaffected.
- `isExcluded` tokens are still fully resolved internally — only export skips them.
- Platform sync for the semantic layer doesn't exist at the bridge level yet: Figma/Penpot could alias semantic tokens to primitive variables/tokens; Sketch/Framer have no aliasing concept and would need a flat naming convention instead, mirroring how their primitive sync already degrades relative to Figma/Penpot.

## Acceptance criteria

- [ ] Given a taxonomy with zero groups, when it's resolved, then the system returns zero tokens without erroring.
- [ ] Given a taxonomy path with no binding, when the system is resolved, then that path still appears in the output with a null value, rather than being omitted.
- [ ] Given a binding whose shade doesn't exist in one theme, when the system is resolved, then only that theme resolves to null for that token — other themes resolve normally.
- [ ] Given a token marked `isExcluded`, when the system is resolved internally, then its value is still computed; when code/tokens are exported, then it's omitted from the output.
- [ ] Given the design-tool UI (once built), when a user picks a taxonomy pattern and gets binding suggestions, then the flow matches the orchestrator's guided flow (propose → refine → confirm) rather than requiring bindings to be typed by hand from scratch.
- [ ] Given the design-tool UI (once built), when a user views the resolved system, then they see the full token matrix (path × theme → shade) without leaving the plugin.

## Out of scope

- Building the design-tool UI itself — this spec scopes it but doesn't implement it.
- Syncing the semantic layer to any platform's native mechanism — not scoped at the bridge level yet, for any platform.
- Deciding whether the design-tool UI allows hand-editing bindings, or stays guided-only — open question, not yet decided (see below).

## Implementation notes

See [`../01-domain-model/color-system.md`](../01-domain-model/color-system.md) for the full data shape (`SystemConfiguration`, `SystemData`, taxonomy schema, binding format) — not duplicated here.

- **Bridges**: none yet for the semantic layer specifically — primitive sync bridges are unaffected until platform sync is scoped.
- **API / MCP**: already covered by `get_color_system`, and by `generate_code` when a system is passed alongside `base`/`themes`.
- **Platforms**: not yet available anywhere as a design-tool artifact — only as generated code/tokens.
- **Credits / plan**: not documented yet — to confirm whether resolving/exporting a Color System consumes credits the same way palette generation does.

## Open questions

- Does an `isExcluded` token still get its per-theme overrides resolved, or should resolution short-circuit for it?
- What should happen in-app when a taxonomy references a palette that later loses a color or a shade a binding points to — surfaced as a warning, silently nulled (current API behavior), or blocking?
- Should the design-tool UI allow editing bindings by hand, or stay guided-only (propose → refine → confirm) like the orchestrator?
- How does semantic sync map onto each platform's native capabilities (aliasing on Figma/Penpot vs. flat naming on Sketch/Framer)?

## See also

- [Color System (domain model)](../01-domain-model/color-system.md) — taxonomy/binding shape, `get_color_system` contract
- [Palette](../01-domain-model/palette.md) — the primitives this layer resolves against
- [Platform bridges](../03-platform-bridges/bridge-actions.md) — where semantic sync would eventually plug in, alongside the existing primitive sync
- [MCP tools](../04-contracts/mcp-tools.md) — `get_color_system`, and `generate_code` with a system attached

## History

| Date | Change |
| ---- | ------ |
| 2026-07-30 | Created — functional spec covering the existing engine/API/MCP behavior and scoping the not-yet-built design-tool UI |
| 2026-07-30 | Internal links added |
| 2026-08-03 | Reformatted to the Problem/User flow/Rules/Acceptance criteria template |
