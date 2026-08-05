# specs-ui-color-palette

Internal engineering specifications for the **UI Color Palette** product, meant to serve as a foundation for further development (new features, refactors, new platform surfaces).

## This folder is not `docs-ui-color-palette`

`docs-ui-color-palette` is the **public/user-facing documentation**, organized by surface (`api/`, `claude/`, `figma/`, `framer/`, `mcp/`, `penpot/`, `sketch/`, `legal/`). It explains *how to use* the finished product.

`specs-ui-color-palette` is the **engineering documentation**, organized by functional domain. It explains *how the product is built* and *what we plan to change*. Audience: the dev team (you, and any future contributor or agent who needs to implement a feature without rediscovering the whole monorepo).

## Method: foundation + specs on demand

We don't specify everything at once. Two content categories, two writing cadences:

1. **Foundation (`00-overview/`, `01-domain-model/`)** — written once, rarely updated. Describes the overall architecture and the domain model (Palette, Color System, User Context). This is the shared reference for all future feature specs.
2. **Feature specs (`02-features/`, `03-platform-bridges/`, `04-contracts/`)** — written one at a time, **right before or during** the relevant development, using the [`TEMPLATE.md`](./TEMPLATE.md) template. Files not yet addressed already exist as *stubs* (summary + pointers to existing code) so we don't start from scratch the day we fill them in.

A feature spec is never an exhaustive audit of existing code: it documents the current state *relevant to the feature at hand*, then the proposal. Nothing more.

## Spec status

Each feature/contract file carries a status in its header:

- `Stub` — summary + code pointers, not yet written
- `Draft` — being written
- `Proposed` — ready for review before implementation
- `Accepted` — validated, ready to guide development
- `Implemented` — the described development has shipped
- `Superseded` — replaced by a newer spec (link to the new one)

## Structure

```
00-overview/          # Monorepo map, glossary — stable
01-domain-model/       # Palette, Color System, User Context — stable
02-features/            # One spec per UI module
03-platform-bridges/    # Bridge contracts + per-platform specifics (Figma, Penpot, Sketch, Framer)
04-contracts/           # External surfaces consumed/exposed: API, MCP, iframe↔host messages
TEMPLATE.md             # Template for any new feature spec
```

## Map

Every file links back here and sideways to what it depends on — start anywhere and follow the "See also" section at the bottom of each doc, or use this index.

### 00 — Overview

- [Architecture](00-overview/architecture.md) — the ~13-repo monorepo map, dependency graph, the two ways to drive the product
- [Glossary](00-overview/glossary.md) — every domain term, each pointing to the spec that defines it in depth

### 01 — Domain model (stable foundation)

- [Palette](01-domain-model/palette.md) — generation inputs, edit operations, local vs. published lifecycle, quotas, export
- [Color System](01-domain-model/color-system.md) — the semantic layer on top of a palette: taxonomy, bindings, resolution
- [User context](01-domain-model/user-context.md) — identity/session, plan/trial/credits, consent, pre-action guardrails

### 02 — Features (one per UI module)

- [Palettes](02-features/palettes.md) — local palette listing, creation, duplication, deletion
- [Colors](02-features/colors.md) — source-color editor (name, hex/LCH, hue/chroma shift, alpha, description)
- [Themes](02-features/themes.md) — theme list, the "active theme" mechanism read by Scale/Preview/Settings
- [Scale](02-features/scale.md) — presets, raw vs. contrast-ratio editing, custom stops, easing
- [Hue/Chroma Distribution](02-features/hue-chroma-distribution.md) — non-linear (`LINEAR`/`HYPERBOLA`/`FREE`) shift curves, extending Scale's and Colors' hue/chroma shift
- [Preview](02-features/preview.md) — contrast scoring, the per-color contrast report, preview settings
- [Settings](02-features/settings.md) — palette name/description, global color settings, deletion
- [Preferences](02-features/preferences.md) — language, deep sync with the host editor
- [Modals](02-features/modals.md) — publishing, pricing/license, onboarding, utility dialogs
- [Color System (feature)](02-features/color-system.md) — what exists today (engine/API/MCP) vs. the not-yet-built design-tool UI
- [Export](02-features/export.md) — the Export mode: code/token export across ~20 formats
- [Inspect](02-features/inspect.md) — the Inspect mode's read-only contrast report
- [Actions](02-features/actions.md) — the top bar and Edit/Inspect/Export mode switcher
- [Creation](02-features/creation.md) — overview of the four alternative ways to seed a new palette
- [Color wheel](02-features/color-wheel.md), [AI generation](02-features/ai-generation.md), [Image extraction](02-features/image-extraction.md), [ColourLovers import](02-features/colourlovers-import.md) — the four methods themselves

### 03 — Platform bridges

- [Bridge catalog](03-platform-bridges/bridge-actions.md) — the shared UI↔host message-to-bridge inventory
- [Figma / FigJam](03-platform-bridges/figma.md)
- [Penpot](03-platform-bridges/penpot.md)
- [Sketch](03-platform-bridges/sketch.md)
- [Framer](03-platform-bridges/framer.md)

### 04 — Contracts (external surfaces)

- [REST API](04-contracts/api-endpoints.md) — `api-ui-color-palette`, all `/v1` endpoints
- [MCP server](04-contracts/mcp-tools.md) — `mcp-ui-color-palette`, tool-to-endpoint mapping
- [Bridge messages & analytics events](04-contracts/events-messages.md) — UI↔host contract and Mixpanel event catalog

## Content origin

The foundation (00-overview, 01-domain-model) was reconstructed on 2026-07-27 by reading the `ui-ui-color-palette` code (types, bridges, stores) and the READMEs of `engine-ui-color-palette`, `api-ui-color-palette`, `mcp-ui-color-palette`. This is a reconstruction, not a design — some points are inferences and must be validated by the author (marked `⚠️ to verify`).

`TEMPLATE.md` was revised on 2026-08-03 to a Problem / User flow / Rules / Acceptance criteria / Out of scope / Implementation notes shape. The seven `02-features/` files existing at the time were converted to match on the same date: as-is behavior now reads as `User flow` + `Rules`, confirmed bugs/gaps are captured as failing `Acceptance criteria` (with a note on why they're not yet met), and settled author answers were folded directly into `Rules`/`Out of scope` instead of staying in a resolved Q&A list. Two sections outside the strict template are kept where they earn their place: `Open questions` (only when something genuinely unresolved remains) and `See also` (cross-links between specs).

Between 2026-08-04, ten more `02-features/` files were added the same way (as-is consolidation from reading the corresponding code), closing gaps identified by cross-referencing the existing specs against the actual `src/ui/` module tree: [Colors](02-features/colors.md), [Themes](02-features/themes.md), [Export](02-features/export.md), [Inspect](02-features/inspect.md), [Actions](02-features/actions.md), [Creation](02-features/creation.md), [Color wheel](02-features/color-wheel.md), [AI generation](02-features/ai-generation.md), [Image extraction](02-features/image-extraction.md), [ColourLovers import](02-features/colourlovers-import.md). Known remaining gaps: the Properties read-only summary tab (Inspect), the Imports context, and the community/published-palette browser (`RemotePalettes.tsx` — not to be confused with the ColourLovers import service, see that spec's Problem section).
