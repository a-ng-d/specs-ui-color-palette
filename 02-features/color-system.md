# Feature — Color System (semantic layer)

- **Status**: Draft — engine/API/MCP available today; the design-tool UI is now named and scoped as its own spec, see [Structure](structure.md)
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

1. **Confirmed for v1**: in-app, the user defines a taxonomy and assigns every binding **by hand** (dropdown ref picker) — not the orchestrator's guided propose → refine → confirm flow. A lighter-weight assist is planned as a later, non-v1 enhancement: once a color is picked for one binding, suggest an adjacent shade (darker/lighter, same color family) as a starting point for related bindings (e.g. a `hover`/`pressed` state next to a `default` one) — not a full taxonomy-pattern/binding-suggestion flow like the orchestrator's.
2. The user can view the resulting token matrix (path × theme → shade) directly in the plugin — see [Structure](structure.md)'s Visualize tab (a read-only dependency diagram, not a literal matrix table).
3. The user syncs the semantic layer into the platform's native mechanism, same as primitives already sync today — except on Sketch/Framer, where there's no aliasing capability at all, so sync there produces organizational structure only (see Rules).

## Rules

- The Color System is additive and non-destructive: building or changing it never touches the underlying palette.
- A taxonomy with zero groups produces zero tokens — not an error, just an empty system.
- A token with no matching binding still appears in the output, with a null value per theme, instead of being silently dropped — so a taxonomy can be defined before every path is bound, and gaps stay visible.
- A binding whose shade reference doesn't exist for a given theme (e.g. a theme that dropped a color the others have) resolves to null for that token in that theme only; other themes are unaffected.
- `isExcluded` tokens are **never resolved at all** — confirmed engine behavior (`System.resolveToken()` forces a null `shadeId` for every theme when `isExcluded` is true, the same path as "no binding exists"), not "resolved internally and only hidden at export" as previously stated here. An excluded token therefore never appears in code, variables, tokens, any export, or generated documentation.
- **A binding that can't resolve (primitive not linked) — confirmed API behavior**: the API doesn't block or throw; it keeps returning the unresolved/null value for that token, the same as any other broken reference (see the "resolves to null" rule above) — it's not repaired or hidden automatically. Repairing it is left to other tooling: in-app (design-tool UI), the affected token is tagged with a warning (see [Structure](structure.md)); for a cloud-published palette, the expectation is that MCP-driven tools (e.g. an LLM orchestrator) are used to update/rebind the palette's system, rather than the bare API doing anything special.
- Platform sync for the semantic layer doesn't exist at the bridge level yet. Figma/Penpot can alias semantic tokens to primitive variables/tokens (a real reference). **Sketch and Framer have no aliasing concept at all** — there's no binding/reference mechanism to sync there; what gets created instead is purely **organizational structure** (a "Primitives" + a "Semantics" folder/group of otherwise-standalone styles) — the semantic *role* is conveyed through naming and folder structure, not through any live link to the primitive value. See [Structure](structure.md) for the per-platform detail (Figma/Penpot/Sketch/Framer) this generalizes.

## Acceptance criteria

- [ ] Given a taxonomy with zero groups, when it's resolved, then the system returns zero tokens without erroring.
- [ ] Given a taxonomy path with no binding, when the system is resolved, then that path still appears in the output with a null value, rather than being omitted.
- [ ] Given a binding whose shade doesn't exist in one theme, when the system is resolved, then only that theme resolves to null for that token — other themes resolve normally.
- [ ] Given a token marked `isExcluded`, when the system is resolved, then it resolves to a null value for every theme (nothing is computed for it) — and consequently it's omitted from code, variables, tokens, any export, and generated documentation.
- [ ] Given the design-tool UI (once built), when a user assigns a binding, then it's done by hand (v1) — no orchestrator-style propose/refine/confirm flow is required or expected.
- [ ] Given the design-tool UI (once built), when a user views the resolved system, then they see it as a read-only dependency diagram without leaving the plugin (not a literal path × theme matrix table — see [Structure](structure.md)).
- [ ] Given a binding that can't resolve (its primitive isn't linked), when viewed in the design-tool UI, then the affected token is tagged with a warning; the underlying API keeps returning the unresolved value rather than blocking or silently repairing it.
- [ ] Given a platform with no aliasing capability (Sketch, Framer), when the semantic layer is synced, then only organizational structure (folders/naming) is created — no live reference/binding to the primitive value is possible there.

## Out of scope

- Building the design-tool UI itself — this spec scopes it but doesn't implement it.
- Syncing the semantic layer to any platform's native mechanism — not scoped at the bridge level yet, for any platform.
- The adjacent-shade suggestion enhancement (darker/lighter, same family) — confirmed as a **future**, non-v1 addition on top of the confirmed hand-editing v1 (see User flow); not scoped further here.

## Implementation notes

See [`../01-domain-model/color-system.md`](../01-domain-model/color-system.md) for the full data shape (`SystemConfiguration`, `SystemData`, taxonomy schema, binding format) — not duplicated here.

- **Bridges**: none yet for the semantic layer specifically — primitive sync bridges are unaffected until platform sync is scoped.
- **API / MCP**: already covered by `get_color_system`, and by `generate_code` when a system is passed alongside `base`/`themes`.
- **Platforms**: not yet available anywhere as a design-tool artifact — only as generated code/tokens.
- **Credits / plan**: not documented yet — to confirm whether resolving/exporting a Color System consumes credits the same way palette generation does.

## Open questions

- ~~Does an `isExcluded` token still get its per-theme overrides resolved, or should resolution short-circuit for it?~~ — resolved (2026-08-05): resolution short-circuits; see Rules and Acceptance criteria above.
- ~~What should happen in-app when a taxonomy references a palette that later loses a color or a shade a binding points to — surfaced as a warning, silently nulled, or blocking?~~ — resolved (2026-08-05): silently nulled at the API/engine level (unchanged, not repaired automatically); tagged with a warning in the design-tool UI; expected to be fixed via MCP-driven tooling (e.g. an LLM orchestrator) for cloud-published palettes. See Rules.
- ~~Should the design-tool UI allow editing bindings by hand, or stay guided-only (propose → refine → confirm) like the orchestrator?~~ — resolved (2026-08-05): hand-editing for v1, with a lighter-weight adjacent-shade suggestion planned as a future (non-v1) enhancement. See User flow and Out of scope.
- ~~How does semantic sync map onto each platform's native capabilities (aliasing on Figma/Penpot vs. flat naming on Sketch/Framer)?~~ — resolved (2026-08-05): Figma/Penpot alias to primitives (a real reference); Sketch/Framer have no binding/aliasing at all — sync there produces organizational structure only (folders/naming), conveying the semantic role without a live link. See Rules and [Structure](structure.md) for the full per-platform breakdown.

*(none remaining — all open questions from this spec have been resolved as of 2026-08-05)*

## See also

- [Structure](structure.md) — the named, concretely-scoped design-tool UI for this semantic layer (mode switcher entry, taxonomy editor, bindings, local/cloud persistence gap)
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
| 2026-08-05 | Design-tool UI named "Structure" and split into its own spec — see [Structure](structure.md) |
| 2026-08-05 | Corrected the `isExcluded` behavior: the engine never resolves an excluded token at all (confirmed by the requester as intended), not "resolved internally, hidden only at export" as previously documented here and in the domain model. Resolves a prior open question. |
| 2026-08-05 | Resolved the remaining three open questions: broken-reference behavior (silently nulled at API/engine level, warning-tagged in the design-tool UI, expected to be fixed via MCP-driven tooling for cloud palettes); hand-editing confirmed for v1 with an adjacent-shade suggestion as a future enhancement (superseding the orchestrator-guided-flow framing in User flow); per-platform semantic sync confirmed as real aliasing on Figma/Penpot vs. structure-only (no binding at all) on Sketch/Framer. No open questions remain in this spec. |
