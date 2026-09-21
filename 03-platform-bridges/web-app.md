# Platform — Web App

- **Status**: **Implemented** — everything platform-level (routing/SSR, service-component reuse, local storage, the sharing-link mechanism) is confirmed live, by reading `web-ui-color-palette` source directly and by the requester (2026-09-21). Still Draft/unverified: the "iso editing experience" claim below (feature-for-feature parity with the plugins hasn't been independently checked file-by-file) and [Team spaces](../02-features/team-spaces.md) (not started).
- **Host repo**: `web-ui-color-palette` (already listed in [`00-overview/architecture.md`](../00-overview/architecture.md) as a "Standalone web application", currently undocumented beyond that one line)
- **Related spec**: [Sharing links](../02-features/sharing-links.md), [Team spaces](../02-features/team-spaces.md), [Palette](../01-domain-model/palette.md) §6 (publication lifecycle, reused as-is), [Modals](../02-features/modals.md) (Publication modal, reused as-is)

## Objective

Organic growth channel for Yelbolt: a public, no-install surface that lets anyone try the palette-editing experience directly in a browser, funneling into the design-tool plugins and paid plans rather than replacing them.

## What's iso vs. what's different vs. the plugins

- **Iso**: the palette-editing experience itself — creation, source colors, themes, scale, settings, export, inspect, structure/color-system, publication — is meant to match the design-tool plugins feature-for-feature. No new editing capability is introduced by the Web App itself.
- **Different — UI skin**: the plugin chrome (Figma/Penpot/Sketch/Framer-flavored panel) is replaced by a Yelbolt-branded UI proper to the Web App. **Resolved by code** (see Routing & SSR below): it's a themed reuse of `ui-ui-color-palette`, the same way each design-tool host already swaps `colorMode` (e.g. `figma-light`/`figma-dark`) — each Web App page mounts the matching `ui-ui-color-palette` service component directly. [`00-overview/architecture.md`](../00-overview/architecture.md)'s existing dependency graph, which shows `web-ui-color-palette` depending on the engine and API only, is therefore out of date and needs correcting to also show the `ui-ui-color-palette` dependency.
- **Different — no design-tool sync**: none of the platform-specific bridges documented for Figma/Penpot/Sketch/Framer apply here — no variable/token/style sync, no on-canvas document generation. There is no design-tool document to sync to or store the palette in.
- **New — sharing by link, implemented**: see [Sharing links](../02-features/sharing-links.md) — full mechanism confirmed, see the Routing & SSR and Anchor points sections below.
- **New — team spaces, not started**: see [Team spaces](../02-features/team-spaces.md).
- **Account model**: editing a palette never requires an account (same "try before you sign up" philosophy as the plugins' local palettes) — an account is required only to **publish** a palette and to **subscribe** (paid plan / team seats).

## Routing & SSR

Implementation confirmed by reading `web-ui-color-palette` (`src/entry-server.tsx`/`entry-client.tsx`, `src/pages/*.tsx`, `src/data/webConfig.ts`). The Web App is server-rendered and exposes each of the plugins' top-level services as its own URL-addressable route, rather than an in-app nav-only switch:

| Route | Service (`App.tsx`'s `Service` state, per [Creation](../02-features/creation.md)) | Purpose |
| --- | --- | --- |
| `/manage` | `MANAGE` (default) | Palette management — browse/create/open, the default landing surface. Also the resolution target for share links (see below). |
| `/gen` | `GEN` | AI color generation — see [AI generation](../02-features/ai-generation.md). |
| `/extract` | `EXTRACT` | Dominant-color extraction from an uploaded image — see [Image extraction](../02-features/image-extraction.md). |
| `/wheel` | `WHEEL` | Color-harmony creation via the color wheel — see [Color wheel](../02-features/color-wheel.md). |
| `/explore` | `EXPLORE` | Third-party palette import used to seed a new palette — see [ColourLovers import](../02-features/colourlovers-import.md). **Naming caution carried over from that spec**: this is *not* the community/published-palette browser (`RemotePalettes.tsx`, still undocumented) — it's one of the four alternative creation methods. **Provider transition, confirmed by the requester**: this route already runs against **Color Hunt** instead of ColourLovers on the Web App — single-select hue filter (no longer cumulative) and results ordered by most-recently-created (no more vote-based ordering, since Color Hunt doesn't expose vote counts). Design-tool plugins still use the original ColourLovers integration; renaming the still-ColourLovers-named code identifiers (fee key, analytics event, `source` tag, module/file name) is pending. Full detail in [ColourLovers import](../02-features/colourlovers-import.md#planned-change--color-hunt-replaces-colourlovers). |

Each page mounts the matching `ui-ui-color-palette` service component (`WithConfig`/`WithTranslation`-wrapped, e.g. `ManagePalette`, `Explore`) inside a shared `AppStateContext` — this is the "themed reuse of `ui-ui-color-palette`" branch of the Open questions below, now confirmed by code rather than open: the Web App imports the plugin's own service components directly, it does not re-implement them.

## Anchor points to confirm once implementation starts

- ~~Possible `Editor`/`Platform` value: `web`~~ — **resolved**: `webConfig.ts` already sets `env.editor: 'web'` and `env.platform: 'yelbolt'`. Still not reflected in [`00-overview/glossary.md`](../00-overview/glossary.md)'s documented `Platform`/`Editor` enums (currently `figma | penpot | sketch | framer`) — that doc needs updating to match.
- ~~Storage of a **local** (unpublished, account-free) palette~~ — **resolved**, confirmed by the requester: `src/data/bridge/db.ts` (`getPalette`/`setPalette`) backs onto **IndexedDB**, not `localStorage` — a genuinely new storage location, not a reuse of `stores/localPalettes.ts`'s document-backed model. `localStorage` is deliberately the plugins' pattern instead (recommended for their sandboxed host environment); the Web App uses IndexedDB as the sturdier, more secure option for an unsandboxed browser environment.
- ~~Whether `web-ui-color-palette` re-implements each spec independently, or imports `ui-ui-color-palette`~~ — **resolved**, see Routing & SSR above: it imports the shared service components.

## Out of scope (for this platform, at least for v1)

- Any design-tool sync bridge (variables, styles, tokens, document generation) — see [Bridge catalog](bridge-actions.md).
- Anything requiring a design-tool host API — this platform only ever talks to the engine/API, never to Figma/Penpot/Sketch/Framer.

## Open questions

- ~~Reuse `ui-ui-color-palette` (themed) vs. a separate implementation~~ — resolved by code, see Routing & SSR: it reuses the shared service components. `00-overview/architecture.md`'s dependency graph (showing `web-ui-color-palette` depending only on the engine/API) needs correcting to also show the `ui-ui-color-palette` dependency.
- ~~Exact local (account-free, unpublished) storage mechanism for a Web App palette~~ — resolved: IndexedDB, see Anchor points above.
- ~~Whether `Platform`/`Editor` gain a `web` value now~~ — resolved: `editor: 'web'` already in use in `webConfig.ts`; glossary doc still needs syncing.
- ~~Exact scope of the ColourLovers → Color Hunt provider transition on `/explore`~~ — resolved by the requester, see the Routing & SSR table and [ColourLovers import](../02-features/colourlovers-import.md#planned-change--color-hunt-replaces-colourlovers) for full detail (data shape, filter rework, ordering, rollout, and the still-pending code-identifier rename).

## See also

- [`00-overview/architecture.md`](../00-overview/architecture.md) — `web-ui-color-palette`'s existing (thin) entry in the package map
- [Figma](figma.md), [Penpot](penpot.md), [Sketch](sketch.md), [Framer](framer.md) — the design-tool platforms this one deliberately does *not* behave like (no sync, no document storage)
- [Sharing links](../02-features/sharing-links.md), [Team spaces](../02-features/team-spaces.md) — the two genuinely new features this platform introduces
- [Chrome Extension](chrome-extension.md) — the other new platform, which hands off to this one for full editing

## History

_Ordered most recent to oldest._

| Date | Change |
| --- | --- |
| 2026-09-21 | Status changed from Draft to **Implemented** for everything platform-level (routing/SSR, storage, sharing-link mechanism) — the requester confirmed this is live, not a partial/draft implementation. Team spaces and the unverified "iso editing experience" claim remain Draft, called out explicitly in the Status line instead of left implicit. |
| 2026-09-21 | Requester answered all remaining Open questions from the same-day update: Web App local storage confirmed as IndexedDB (vs. `localStorage` for the plugins' sandboxed hosts), and the ColourLovers → Color Hunt transition on `/explore` confirmed in full (see [ColourLovers import](../02-features/colourlovers-import.md)). No open questions remain in this file as of this entry. |
| 2026-09-21 | Added Routing & SSR section (confirmed `/manage`, `/gen`, `/extract`, `/wheel`, `/explore` from `web-ui-color-palette` source), resolved three Anchor points/Open questions by reading code (`editor: 'web'`, shared-component reuse, local-store location), and flagged the requester-announced ColourLovers → Color Hunt provider transition on `/explore`. |
| 2026-08-23 | Created as Draft, consolidated from the requester's description of the Web App as a growth-focused, iso-but-reskinned surface with link sharing and team spaces. No implementation exists yet. |
