# Feature — Creation (how a palette gets seeded)

- **Status**: Implemented
- **Package(s) concerned**: `ui-ui-color-palette`
- **UI module**: `src/ui/App.tsx` (service router), `src/ui/services/ManagePalette.tsx`, and the four creation services documented separately: [Color wheel](color-wheel.md), [AI generation](ai-generation.md), [Image extraction](image-extraction.md), [ColourLovers import](colourlovers-import.md)
- **Related spec**: [Palettes](palettes.md), [Colors](colors.md), [Actions](actions.md)

## Problem

There are five parallel top-level "services" (`App.tsx`'s `Service` state: `MANAGE` | `GEN` | `EXTRACT` | `WHEEL` | `EXPLORE`), switched by a nav bar shown only when more than one is active. `MANAGE` is the default palette-management surface (browse/create/open — see Palettes); the other four are alternative ways to seed a brand-new palette's source colors before handing off to `MANAGE`. This spec is the shared overview; each method's specifics live in its own file.

## User flow

1. A services nav bar (hidden if only one service is plan/platform-active) lets the user switch between: **Manage** (default), **Gen** (AI generation), **Extract** (image color extraction), **Wheel** (color-harmony wheel), **Explore** (ColourLovers import, *migrating to Color Hunt* — see [ColourLovers import § Planned change](colourlovers-import.md#planned-change--color-hunt-replaces-colourlovers) — and *not* the community/published-palette browser, see Rules). On the Web App, these five map 1:1 to the `/manage`, `/gen`, `/extract`, `/wheel`, `/explore` routes — see [Web App](../03-platform-bridges/web-app.md#routing--ssr).
2. Each of the four non-Manage services lets the user produce a candidate set of source colors through its own method, then commit with a single "Use this palette" action.
3. Committing does two things at once: switches the service back to `MANAGE`, and sends the same `CREATE_PALETTE` message every method shares — the palette is created **and immediately opened** (Edit mode), with no separate confirmation step in `ManagePalette` itself.
4. Independently, `ManagePalette`'s own `BROWSE` subservice (the default Palettes screen) has its own, simpler seeding path: it starts with a set of random default source colors (or ones pulled from the current canvas selection), which the user can accept as-is via an explicit "Create palette" button — no separate generation step, no extra fee.

## Rules

- All four alternative methods share the same contract: produce `Array<SourceColorConfiguration>` tagged with a distinct `source` value (`'HARMONY'`, `'AI'`, `'IMAGE'`, `'COLOUR_LOVERS'`, vs. `'DEFAULT'`/`'CANVAS'` for the plain Manage flow), then call the same `CREATE_PALETTE` bridge message.
- All four use a **two-fee cost model**: one fee for the generation/extraction/import step itself (`harmonyCreate`, `aiColorsGenerate`, `imageColorsExtract`, `colourLoversImport`), and a separate `paletteCreate` fee charged when the user actually commits to creating a local palette from the result. This is consistent across all four independent implementations — read as intentional, not a double-charge bug.
- All four independently read a shared `localPalettesCount` prop for their own local-palette-quota check (same pattern already noted in Palettes for Explore/AI/wheel/image import) — none of them own that count themselves.
- Committing from any of the four **creates and opens** the palette in one action — unlike `ManagePalette`'s own default `BROWSE → Create palette` path, there's no intermediate draft state the user can back out of before it's actually created.
- `services/Explore.tsx` is **not** the "Explore palettes" community/published-palette browser referenced in Palettes' empty-state CTA (that's `contexts/RemotePalettes.tsx`, reached via `BrowsePalettes`' own `REMOTE_PALETTES` tab, still undocumented). It's an entirely different feature: importing a palette from the third-party **ColourLovers** API. The name collision is confirmed in code, not a typo — see [ColourLovers import](colourlovers-import.md).

## Acceptance criteria

- [ ] Given any of the four creation services, when the user commits a candidate palette, then exactly one `paletteCreate`-fee debit and one service-specific-fee debit occur, matching the two-step cost model.
- [ ] Given the local-palette quota is reached, when the user tries to commit from any of the four services, then the action is blocked with a trial/upgrade prompt, consistent with the quota enforcement documented in Palettes.
- [ ] Given only one service is active for the current plan/platform, when the app loads, then the services nav bar is hidden entirely (not shown with a single disabled-looking option).

## Out of scope

- The `MANAGE` service itself — fully documented in [Palettes](palettes.md), [Actions](actions.md), and the Edit/Inspect/Export mode specs.
- The community/published-palette browser (`contexts/RemotePalettes.tsx`) reachable from Palettes' "Explore palettes" CTA — not covered here or in ColourLovers import; still an open documentation gap.
- Any change to the two-fee cost model or the quota-check pattern — documented as-is.

## Implementation notes

`App.tsx` owns `state.service` and renders the matching top-level component; each of the four alternative services receives `onChangeService` to hand control back to `MANAGE` on commit. `ManagePalette.tsx` owns the actual palette-store writes on load (`onLoadPalette`) and reset (`onResetPalette`), regardless of which service produced the source colors — see its own `subservice: 'BROWSE' | 'OPEN'` state, which is unrelated to `App.tsx`'s `service` state despite the similar naming.

- **Stores**: `$palette`/`$themes`/`$dates`/`$publicationStatus`/`$creatorIdentity` are all set by `ManagePalette.onLoadPalette` once the host confirms creation — none of the four creation services write to these stores directly.
- **Bridges**: all four send the same `CREATE_PALETTE` message shape (`{ sourceColors, exchange }`); the host's reply eventually reaches `ManagePalette.handleMessage`'s `LOAD_PALETTE` case, which is what actually opens the new palette.
- **Analytics**: each method tracks its own import/generation event (`CREATE_COLOR_HARMONY`, `EXTRACT_DOMINANT_COLORS`, `IMPORT_COLOUR_LOVERS`, plus GenAI's own event — see each method's spec) via `trackImportEvent`, separate from the shared `CREATE_PALETTE` action event.
- **Platforms**: no platform-specific branching identified at this router level.

## Open questions

*(none)*

## See also

- [Palettes](palettes.md) — the `MANAGE` service's default seeding path (random/canvas colors, explicit Create button)
- [Color wheel](color-wheel.md), [AI generation](ai-generation.md), [Image extraction](image-extraction.md), [ColourLovers import](colourlovers-import.md) — the four alternative methods
- [Actions](actions.md) — the analogous mode-switching pattern, one level down, inside an already-open palette

## History

| Date | Change |
| --- | --- |
| 2026-08-04 | Created — as-is consolidation from reading `App.tsx`'s service router and `ManagePalette.tsx`, as an overview for the four newly-documented creation methods |
