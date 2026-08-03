# Feature — ColourLovers import (third-party palette seeding)

- **Status**: Implemented
- **Package(s) concerned**: `ui-ui-color-palette`
- **UI module**: `src/ui/services/Explore.tsx`
- **Related spec**: [Creation](creation.md) (overview, shared cost model), [Palettes](palettes.md)

## Problem

One of four alternative ways to seed a new palette: browse palettes from the third-party **ColourLovers** site and import one. It works as expected. Naming warning: this module is called `Explore.tsx` and reached via the `EXPLORE` top-level service, but it is **not** the "Explore palettes" community/published-palette browser referenced in Palettes' empty-state CTA — that's a different, still-undocumented module (`contexts/RemotePalettes.tsx`). Both happen to be reachable under similar wording; don't conflate them.

## User flow

1. The user browses a paginated list of ColourLovers palettes, optionally filtered by hue (Any, Yellow, Orange, Red, Green, Violet, Blue — multi-select, "Any" clears the rest).
2. Scrolling/paging loads more results (`Load more`) until the source reports no more results (`COMPLETE`) or a fetch fails (`ERROR`, shown with a retry-worthy error message).
3. From a listed palette, the user can either open it directly (jump straight into creating a palette from it) or add its colors — both funnel into the same commit action.
4. Committing does the same handoff as the other three methods (see Creation): the palette is created and opened immediately, tagged with this ColourLovers palette's colors.

## Rules

- Palettes are fetched through a CORS proxy worker (`config.urls.corsWorkerUrl`) wrapping `https://www.colourlovers.com/api/palettes`, paginated by a configured page size (`config.limits.pageSize`).
- Changing the active hue filter resets pagination to page 1 and clears the current results before refetching — not an incremental re-filter of already-loaded results.
- Generated source colors are tagged `source: 'COLOUR_LOVERS'` and, unlike the other three creation methods, are marked `isRemovable: true` and seeded with chroma shift **100** (not 0) — closer to Colors' own "add color" default than to Color wheel/AI/Image's shared 0/0/`isRemovable: false` pattern.
- Unlike the other three methods (which charge their generation/extraction fee as soon as a result is produced), the ColourLovers fee (`colourLoversImport`) is only charged **at commit time**, together with `paletteCreate` — because browsing/filtering the list itself has no per-result cost.

## Acceptance criteria

- [ ] Given the hue filter changes, when new results are requested, then pagination resets to page 1 and previously loaded results are cleared first.
- [ ] Given the ColourLovers API request fails, when the error is caught, then the list shows an error state rather than an empty or infinitely-loading one.
- [ ] Given a palette is committed, when the action completes, then both the `colourLoversImport` fee and the `paletteCreate` fee are debited, in that order.
- [ ] Given the local-palette quota is reached, when the user tries to commit, then the action is blocked with a trial/upgrade prompt.

## Out of scope

- Adding new filters beyond the 6 hue categories.
- Any change to the third-party ColourLovers API integration itself.
- Reconciling this module's naming with the separate "Explore palettes" community browser (`RemotePalettes.tsx`) — flagged, not renamed here.

## Implementation notes

Reuses the engine's `SourceColorConfiguration` shape; results are typed as `ColourLovers` objects from the external API response, mapped to source colors on commit only (not on every list render).

- **Stores**: reads `$palette` only to attach the current in-progress exchange data to the `CREATE_PALETTE` message; writes nothing itself.
- **Bridges**: sends `CREATE_PALETTE` with `{ sourceColors, exchange }` — same shape as the other three creation methods. The palette-list fetch itself is a plain `fetch()` to the CORS proxy, not a plugin-message bridge.
- **Analytics**: `IMPORT_COLOUR_LOVERS` (commit) via `trackImportEvent`; the underlying `CREATE_PALETTE` action itself is tracked separately.
- **Credits**: `colourLoversImport` + `paletteCreate`, both debited at commit time — see Creation's shared cost model and this module's Rules for why the timing differs from the other three.
- **Platforms**: none identified.

## Open questions

*(none)*

## See also

- [Creation](creation.md) — the shared overview, cost model, and the three sibling methods
- [Palettes](palettes.md) — the similarly-named but functionally distinct "Explore palettes" community/published-palette browser

## History

| Date | Change |
| --- | --- |
| 2026-08-04 | Created — as-is consolidation from reading `Explore.tsx`, following up on a request to cover the creation services; naming collision with "Explore palettes" flagged explicitly |
