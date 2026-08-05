# Domain model — Color System (semantic layer)

- **Status**: Implemented
- **Last updated**: 2026-08-05
- **Source**: `engine-ui-color-palette` README (§ Color System & Semantic Tokens), `System` / `Code` classes

## 1. Position in the pipeline

The Color System is built **on top of** an already-resolved palette (`PaletteData`, see `palette.md`). It never modifies the primitives — it adds a layer of semantic names that point to existing shades.

```
Base + Themes → Data.makePaletteData() → PaletteData
                                            │
                          System({ paletteData, system: { schema, bindings } })
                                            │
                                    .makeSystemData() → SystemData
                                            │
                        Code({ paletteData, systemData }).makeCssFiles() → combined files
                                    (primitives --color-blue-500 + semantic --brand-default)
```

## 2. Schema (taxonomy)

```
schema.groups: Array<{
  id: string          # e.g. 'role'
  name: string         # e.g. 'Role'
  members: Array<{ id: string; name: string }>   # e.g. { id: 'brand', name: 'Brand' }
}>
```

A taxonomy has as many groups as desired semantic dimensions (Role, Prominence, State, Surface/Content — the common patterns cited by the Claude orchestrator: *Role × State*, *Role × Prominence × State*, *Surface × Content*, *Custom*).

## 3. Bindings

```
bindings: Array<{
  path: string[]              # path within the taxonomy, e.g. ['brand', 'default']
  description?: string
  ref: string                  # primitive shade reference, format 'colorId:stop', e.g. 'blue:500'
  overrides?: { [themeId]: string }   # alternate ref per theme, e.g. { dark: 'blue:400' }
  isExcluded?: boolean          # if true, the binding is never resolved at all (see below) — not just excluded from output
}>
```

**Correction (2026-08-05)**: `System.resolveToken()` short-circuits on `isExcluded` — when true, every theme's `refs` entry is forced to `{ themeId, shadeId: null }`, the exact same code path as "no binding exists," without parsing `ref`/`overrides` at all. So an excluded token is not "still resolved internally and only hidden at export" (an earlier, incorrect description of this field) — it is **never resolved**, and consequently never appears in code, variables, tokens, any export, or generated documentation. Confirmed as intended engine behavior by the requester (the engine is the source of truth here). See [Structure](../02-features/structure.md) for where this was caught and confirmed.

## 4. MCP / API contract

`get_color_system` (MCP tool) = `POST /v1/get-color-system` (API) — takes `base`, `themes`, `system: { schema, bindings }`, returns `SystemData`. See [`04-contracts/mcp-tools.md`](../04-contracts/mcp-tools.md) and [`04-contracts/api-endpoints.md`](../04-contracts/api-endpoints.md).

`SystemData` then feeds `generate_code` (with `system` in addition to `base`/`themes`) to produce a primitives file + a semantic file per format.

## 5. What's still to be specified (do it when the need arises)

- Binding validation rules (an invalid `ref`, a `path` that matches no member of the schema — current behavior not documented here, to check in `engine-ui-color-palette/src/modules/system/system.ts` the day it needs to evolve).
- ~~Interaction between `isExcluded` and per-theme overrides~~ — resolved (2026-08-05): `isExcluded` short-circuits before any `ref`/`overrides` parsing happens, so overrides are never resolved for an excluded token either. See the correction under §3 above.

## See also

- [Glossary](../00-overview/glossary.md) — Color System terms
- [Palette](palette.md) — the primitives this layer is built on top of
- [Color System (feature)](../02-features/color-system.md) — current behavior, open questions, and the not-yet-built design-tool UI
- [MCP tools](../04-contracts/mcp-tools.md), [REST API](../04-contracts/api-endpoints.md) — the contract this model is exposed through
