# Feature — Team spaces (Web App)

- **Status**: Draft
- **Package(s) concerned**: `web-ui-color-palette` (not implemented yet), `api-ui-color-palette` (new team/seat data model and endpoints, not implemented yet)
- **UI module**: Web App only, proposed `/team` surface
- **Related spec**: [Web App](../03-platform-bridges/web-app.md), [Palette](../01-domain-model/palette.md) §6 (existing `org` visibility scope this may or may not be the real implementation of — see Open questions), [User context](../01-domain-model/user-context.md) (individual `PlanStatus`/credits model, distinct from the seat model this introduces)

## Problem

The Web App's account model today (per the requester's brief and the existing individual `PlanStatus`/credits model in [User context](../01-domain-model/user-context.md)) has no notion of a group of people sharing a pool of palettes. [Palette](../01-domain-model/palette.md) §6 already documents an `org` visibility scope for published palettes ("managed by an organization admin on the remote side") but flags it as unconfirmed from code and with no client-side UI ever built for it. Team spaces are the requester's explicit ask for a user-facing way for several people to access a shared set of palettes, billed per seat.

## User flow

1. An account holder creates a **Team space** (name, at minimum) from the Web App.
2. The team owner/admin invites members; each accepted invite occupies a **seat**.
3. Members access the team's shared palettes from a `/team` surface, alongside their own personal (non-team) palettes.
4. The team is billed on a **seat model** — cost scales with the number of members, distinct from the existing individual Free/Pro plan.

## Rules

- Access to a team's palettes is gated by team membership — knowing a `/team` URL is not sufficient without being an authenticated member (this interacts with [Sharing links](sharing-links.md)'s "no account to view" rule — see Open questions there and here).
- Seats drive billing, but whether adding a member requires an already-purchased free seat vs. auto-expanding the subscription is not specified by the requester (see Open questions).
- Relationship to the existing `org` publish-visibility scope (Palette §6) is unresolved: is a Team space the concrete, user-facing implementation of that previously-unbuilt scope, or a separate, parallel concept that happens to look similar? This is the single most consequential open question here, since building Team spaces without reconciling it risks ending up with two overlapping "group of people share palettes" mechanisms.

## Acceptance criteria

- [ ] Given a signed-in account holder, when they create a Team space, then they become its owner and it starts with just them occupying a seat.
- [ ] Given a Team space, when the owner invites someone and the invite is accepted, then that person occupies an additional seat and gains access to the team's palettes via `/team`.
- [ ] Given a non-member, when they attempt to access a team's `/team` surface, then access is denied.
- [ ] Given a member of a Team space, when they open `/team`, then they see the team's shared palettes alongside (not merged into) their own personal palettes.
- [ ] Given a Team space's member count, then billing reflects the seat model (cost scales with seats) rather than the individual Free/Pro plan.

## Out of scope

- Role-based permissions granularity (admin/editor/viewer) — not mentioned by the requester; a flat owner + members model is assumed for this draft (see Open questions).
- Reconciling this feature with the pre-existing `org` visibility scope's actual implementation — flagged, not resolved, in this pass.
- Team access from the Chrome Extension or design-tool plugins — this spec covers the Web App only.

## Implementation notes

- **Data model**: new `teams` and `team_members`/seats tables (Supabase), not present today — `api-ui-color-palette`'s existing palette tables (see [Structure](structure.md)'s Implementation notes for the current fixed column list) have no team/org foreign key today beyond the unconfirmed `MEMBER | ORG` publishing-account-type field noted in [Palette](../01-domain-model/palette.md) §6.
- **API**: new endpoints for team creation, invites, membership, and a team-scoped palette listing — none exist today.
- **Web App**: new `/team` (list) and likely `/team/:id` routes, plus an invite-acceptance flow.
- **Billing**: seat-based subscription, distinct from the existing `storeProWeekId`/`storeProMonthId`/`storeProYearId`/`storeProLifetimeId` individual plan IDs in `Config.plan` (see [User context](../01-domain-model/user-context.md) §2) — likely a new set of plan/price IDs, not a reuse of the existing ones.

## Locales

| Text | Where | Tolgee key | Status |
| --- | --- | --- | --- |
| Team creation / invite flow copy | Web App, `/team` | — | Unread — no mockup yet |

## See also

- [Web App](../03-platform-bridges/web-app.md) — the only platform this applies to
- [Palette](../01-domain-model/palette.md) §6 — the existing, unconfirmed `org` visibility scope this may overlap with
- [User context](../01-domain-model/user-context.md) — the individual plan/credits model this sits alongside, not inside
- [Sharing links](sharing-links.md) — a possible additional transport for a team palette, interaction unresolved

## Open questions

- Is Team space the real, finally-built implementation of Palette §6's `org` visibility scope, or a separate concept?
- Role granularity beyond owner/member?
- Seat billing mechanics: pre-purchased seats vs. auto-scaling subscription on invite acceptance?
- Can a team palette be published to Community, and if so under which creator identity?
- Can a team palette be shared via a [Sharing link](sharing-links.md), and if so, does opening it still require team membership?

## History

| Date | Change |
| --- | --- |
| 2026-08-23 | Created as Draft, consolidated from the requester's description of Team spaces with a seat-based billing model for the Web App. Flagged a significant open question: the relationship to Palette §6's pre-existing, never-built-on `org` visibility scope. No implementation exists yet. |
