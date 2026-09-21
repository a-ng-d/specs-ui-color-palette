# Feature — ColourLovers import (third-party palette seeding)

- **Status**: **Forked by platform, confirmed by the requester 2026-09-21.** The Web App's `/explore` already runs against **Color Hunt**; design-tool plugins still run the **ColourLovers** flow described below in User flow/Rules/Acceptance criteria. Full Color Hunt detail in [Planned change](#planned-change--color-hunt-replaces-colourlovers) below — that section is no longer a future intent for the Web App, it's what's live there today; it remains a to-do for the plugins.
- **Package(s) concerned**: `ui-ui-color-palette`
- **UI module**: `src/ui/services/Explore.tsx`, exposed on the Web App as the `/explore` route (see [Web App](../03-platform-bridges/web-app.md#routing--ssr))
- **Related spec**: [Creation](creation.md) (overview, shared cost model), [Palettes](palettes.md), [Web App](../03-platform-bridges/web-app.md) (`/explore` route)

## Problem

One of four alternative ways to seed a new palette: browse palettes from a third-party site and import one. It works as expected — **on design-tool plugins this is still the ColourLovers flow described below; on the Web App it's already the Color Hunt flow described in [Planned change](#planned-change--color-hunt-replaces-colourlovers)**. Naming warning: this module is called `Explore.tsx` and reached via the `EXPLORE` top-level service, but it is **not** the "Explore palettes" community/published-palette browser referenced in Palettes' empty-state CTA — that's a different, still-undocumented module (`contexts/RemotePalettes.tsx`). Both happen to be reachable under similar wording; don't conflate them.

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

## Planned change — Color Hunt replaces ColourLovers

**Confirmed by the requester, 2026-09-21 — live on the Web App today, still pending for the design-tool plugins.** Source: `https://www.colourlovers.com/api/palettes` calls are being replaced by **Color Hunt** as the third-party feed for this creation method. This is a genuine rework of the data flow and the filter model, not a drop-in API swap — code identifiers still say ColourLovers, and my source read (grep across `ui-ui-color-palette` and `web-ui-color-palette` for "Color Hunt"/"colorhunt") found none of them renamed yet. Read that as: the *data source* has moved, the *code names* haven't caught up — see Renaming below.

**Data flow — reworked, not a straight swap:**
- ColourLovers today: one paginated list call per page, returning enough per palette to render the grid directly.
- Color Hunt: a **feed** call returns the latest palettes, then a separate **unit request per palette** to fetch that palette's own detail. Two request shapes where there used to be one — the list-rendering code most likely needs the same two-step fetch, not just a different URL.

**Fields available — narrower than ColourLovers:**
- Lost: vote count, and any user/author attribution — Color Hunt's feed/unit-fetch flow doesn't expose either.
- Kept/new: the colors themselves, and a **like** count (replacing votes conceptually, but not the same metric — no evidence it's used for filtering/sorting, just captured).
- Palette name: **not** supplied by the source at all — generated client-side from the palette's own dominant colors, the same pattern other color-less-sourced creation methods already use for naming.

**Hue filter — reworked from cumulative to exclusive:**
- ColourLovers today (see Rules below): multi-select across 6 hues plus "Any", cumulative.
- Color Hunt: **single-select only** — pick exactly one color filter (e.g. red, or yellow, or blue), no combining. "Ça touche, ce n'est plus cumulatif" — confirmed as a filter-model change, not just new hue names.

**Ordering — reworked from popularity to recency:**
- Color Hunt results are sorted by **most recently created**, replacing whatever popularity/vote-based ordering ColourLovers offered — consistent with votes no longer being available at all.

**Renaming — confirmed as necessary, but not yet scheduled:** `source: 'COLOUR_LOVERS'` (on generated source colors), the `colourLoversImport` fee key, the `IMPORT_COLOUR_LOVERS` analytics event, and this module/spec's own naming (`colourlovers-import.md`, `Explore.tsx`) all need to move to Color Hunt naming — the requester confirmed this explicitly ("c'est quelque chose qui va falloir faire effectivement par la suite") as planned but not timed. Until then, code and this doc keep the ColourLovers-derived identifiers as the internal names for a data source that's already changed underneath them on the Web App.

**Rollout:** the Web App already runs the Color Hunt flow described above — this is not a future state for that platform. Design-tool plugins still run the ColourLovers flow as documented in User flow/Rules/Acceptance criteria above; migrating them is separate, scheduled work. No word yet on whether existing palettes already tagged `source: 'COLOUR_LOVERS'` need any data migration, or simply stay as historical data under the old tag once the rename happens.

**Scope:** confirmed as concerning `Explore.tsx`/the Explore creation method generally ("oui, essentiellement, ça concerne Explore"), not narrowly the Web App's `/explore` route — consistent with the module being shared across platforms (see Creation's package listing). The current per-platform split (Web App live, plugins pending) is a rollout-order fact, not a scope boundary.

Still genuinely open, not addressed by any answer so far:

- Whether the fee model changes given the new two-request (feed + per-palette unit fetch) flow — does `colourLoversImport`/its Color Hunt successor still charge once at commit, or does the extra network shape change the cost story?
- Exact mechanism behind the Web-App-live / plugins-pending split: a shared `Explore.tsx` reading a per-platform config value (e.g. a different `corsWorkerUrl` target per platform), a feature flag, or a code fork — not verified in source; relevant to estimating how much work the plugin migration actually is.
- Whether the previous 6 named hue categories (Yellow, Orange, Red, Green, Violet, Blue) carry over as the single-select option set, or Color Hunt's own categorization replaces them.

## Open questions

*(see Planned change above for what's still open on the Color Hunt migration; everything else in this spec is settled as of the 2026-09-21 requester Q&A round)*

## See also

- [Creation](creation.md) — the shared overview, cost model, and the three sibling methods
- [Palettes](palettes.md) — the similarly-named but functionally distinct "Explore palettes" community/published-palette browser
- [Web App](../03-platform-bridges/web-app.md) — where this module is exposed as the `/explore` route, and where the Color Hunt transition was first mentioned

## History

_Ordered most recent to oldest._

| Date | Change |
| --- | --- |
| 2026-09-21 | Requester answered the Planned change open questions in full: confirmed the Web App already runs Color Hunt live (plugins still pending), detailed the reworked data flow (feed + per-palette unit fetch, no votes/user attribution, like count, client-generated names), the filter rework (cumulative → single-select), the recency-based ordering, and that identifier renaming is confirmed-but-unscheduled. Status line updated to reflect the per-platform fork. Three new narrower open questions surfaced (fee model under the new flow, the exact platform-split mechanism, whether the hue category names carry over). |
| 2026-09-21 | Added Planned change section: requester announced ColourLovers → Color Hunt provider migration for the Web App's `/explore` route. No code changes read yet (`Explore.tsx`, `webConfig.ts` fee key, analytics event, and this file's own naming are all still ColourLovers-named) — tracked as an open migration, not a rename. Cross-linked from and to [Web App](../03-platform-bridges/web-app.md#routing--ssr). |
| 2026-08-04 | Created — as-is consolidation from reading `Explore.tsx`, following up on a request to cover the creation services; naming collision with "Explore palettes" flagged explicitly |
