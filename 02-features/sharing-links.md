# Feature — Sharing links (Web App)

- **Status**: **Implemented** — confirmed by reading `web-ui-color-palette` source (`src/data/shareLink.ts`, `src/data/urlPalette.ts`, `src/data/bridge/creations/createPaletteFromLink.ts`, `src/ui/useSyncPaletteUrl.ts`, `src/pages/manage.tsx`) and by the requester answering every design open question directly (2026-09-21). One acceptance criterion still lacks a written test (see Acceptance criteria) — a test-coverage gap, not an open design decision.
- **Package(s) concerned**: `web-ui-color-palette` (implemented), `api-ui-color-palette` (id-resolution goes through `webConfig.dbs.palettesDbViewName` via Supabase directly, not the documented `GET /v1/get-published-palette/:id` REST endpoint — see Implementation notes)
- **UI module**: Web App only — no design-tool plugin equivalent (see [Web App](../03-platform-bridges/web-app.md))
- **Related spec**: [Web App](../03-platform-bridges/web-app.md), [Palette](../01-domain-model/palette.md) §6 (local/published lifecycle and visibility this reuses for the id-bearing case), [Modals](modals.md) (Publication modal — the pre-existing publish flow this is deliberately not replacing), [Team spaces](team-spaces.md) (a team palette can plausibly also be shared this way — see Open questions)

## Problem

Today a palette is either local to a design-tool document, or explicitly published to the community browser (see [Palette](../01-domain-model/palette.md) §6). There's no lightweight "just send someone a link" option, and no surface that could even open such a link — the Web App is the first place this becomes possible. The requester's brief, now confirmed by code: two separate Web App instances (e.g. two different browsers/computers) are **local and never interact with each other** — the only channel between them is a copied URL. A palette's data can be encapsulated directly in that URL; separately, the URL always also carries the palette's **id**, and if data isn't present (or isn't trusted), that id can point to a hosted (published) palette, which then needs to be checked against the database — does it exist, is it linked to the current viewer as author, and is it shared.

## User flow

There are **two distinct ways** a URL carrying a palette reaches someone else, with different payloads — this distinction was not previously documented and matters for what a recipient can actually open:

1. **Explicit "Copy link" action** (`copyShareLink` in `src/data/shareLink.ts`): available on the palette the user is editing, whether or not they're signed in and whether or not the palette has ever been published. It always builds `${origin}/palettes/<id>?data=<JSON.stringify({ base, themes, meta })>` — both the id **and** the full palette payload, unconditionally. This is the "énormément de données encodées" link: fully self-contained, works for anyone, anywhere, account-free.
2. **Copying the URL straight from the browser's address bar** does *not* reproduce that link. Once a palette is open, `useSyncPaletteUrl` (`src/ui/useSyncPaletteUrl.ts`) rewrites the visible URL via `history.replaceState` down to `/palettes/<id>` only — the `data` param is dropped from what's shown in the address bar. So an address-bar copy only ever carries the id.

Opening either link goes through `resolvePaletteFromUrl` (`src/data/urlPalette.ts`), invoked from `manage.tsx` (the `/palettes/:segment?` page; `:segment` is the palette id) on mount/navigation. Resolution order (each step short-circuits on success):

1. **Local id match**: if `id` is present and `getPalette(id)` finds it in *this browser's own* client-side store, jump straight to it — no network call. (This is what "both are local" means in practice: a palette that already exists on this device, e.g. the same person's own browser, is served from local storage before anything else is tried.)
2. **Embedded data**: if a `data` param is present and parses, the palette is created locally from it directly (`createPaletteFromLink`) — **regardless of whether an id is also present and regardless of that id's remote shared/ownership status**. This runs *before* any database check, not as a fallback after one. In other words: the "Copy link" action's link (case 1 above) never touches the database at all on open.
3. **Remote id lookup** (only reached if there was no usable `data` param): fetches the row matching `id` from `webConfig.dbs.palettesDbViewName` via Supabase and branches:
   - **row exists, `is_shared === true`** → the canonical hosted/published palette is loaded (fresh server data).
   - **row exists, not shared, viewer's `currentUserId === creator_id`** → the author's own private palette is loaded (an owner can always open their own palette via their own id-only link — this is the "l'utilisateur connecté reçoit sa propre palette" case).
   - **row exists, not shared, viewer is not the author** → `'blocked'`: an access-denied message is shown inline on `/palettes/<id>`. There is **no embedded-data fallback** at this point, because step 2 already ran first — a blocked result only happens when the link was id-only to begin with (i.e. an address-bar copy, not a "Copy link" action).
   - **row doesn't exist** → falls through to a not-found error message.
4. If nothing above resolves (no id, no data, or both failed), a `t('error.unfoundPalette')` message is dispatched.

Note: a `'blocked'` result is **not cached** as resolved (`manage.tsx` only records `lastResolvedSearch` on non-blocked outcomes) — if the viewer subsequently signs in as the palette's author, the effect re-runs with the new `currentUserId` and can resolve successfully on the same URL without a reload.

## Rules

- Two payload shapes share the same route (`/palettes/:id`, differing only by the optional `?data=` query string): a **data-bearing** link (self-contained, no DB round-trip, works for anyone) and an **id-only** link (triggers the existence/ownership/shared check). The former is always produced by the explicit "Copy link" action; the latter is what remains once the app's own address-bar rewrite has stripped `data` — see User flow.
- No account is required to **open or edit** from either link shape — consistent with the Web App's general "no account to edit" rule (see [Web App](../03-platform-bridges/web-app.md)). An account is only needed for the id-only, not-shared case, and only to be recognized as the *author* (sign-in doesn't unlock someone else's unshared palette).
- **Confirmed by the requester — the data-bearing link is deliberately non-confidential.** The [Palette](../01-domain-model/palette.md) §6 ownership/visibility model (`creator_id` comparison, `isShared` flag) applies **only to the id-only lookup path**. A data-bearing "Copy link" is, by design, a public link: anyone holding it can open it and add their own local copy, regardless of the palette's `isShared`/publication state or who created it. There is no "close"/restrict concept for this path — the requester's framing: a palette isn't something confidential, the link itself *is* the access control. This is a new **transport** for reaching a palette, not a new publish-visibility state — it deliberately sits outside that model rather than reusing it wholesale.
- **Confirmed by the requester — the id is not created lazily for sharing.** Every palette gets its id at **creation time**, and that same id carries through to publication — generating a share link never mints a new hosted record; it just surfaces the id and data the palette already has.
- **Confirmed — no compression, accepted as-is.** The data payload is plain `JSON.stringify`/`JSON.parse` of `{ base, themes, meta }`, percent-encoded by `URLSearchParams` like any other query value (verified against a real share link from the requester — the `%22`/`%7B` sequences are standard URL-encoding of literal JSON, not a compact/binary format). No compact/compressed serialization is implemented, and none is currently planned — URL-length risk for large palettes (many source colors/themes) is a known, accepted characteristic rather than an open problem.

## Acceptance criteria

- [x] Given a palette already present in this browser's local store (matched by id), when its link is opened, then the palette loads from local storage with no network/database call.
- [x] Given a link carrying a `data` param, when opened, then the palette loads from that embedded data directly, with no database call — even if the link's `id` also happens to point to a private, non-owned, or nonexistent remote palette.
- [x] Given an id-only link (no usable `data` param) whose id exists and is shared, when opened by anyone, then the canonical server-side palette data is loaded.
- [x] Given an id-only link whose id exists and is not shared, when opened by its author (signed in, matching `creator_id`), then the private palette loads.
- [x] Given an id-only link whose id exists and is not shared, when opened by someone other than the author, then access is denied (`'blocked'`, inline warning message) — with no embedded-data fallback, since none was present to begin with.
- [x] Given an id-only link whose id no longer exists, when opened, then the viewer sees a not-found error message.
- [x] Given any share link, when opened, then no account/sign-in is required to view or continue editing the palette.
- [ ] Given a blocked resolution, when the viewer subsequently signs in as the palette's author (without reloading), then resolution is retried and succeeds — confirmed by reading `manage.tsx`'s `lastResolvedSearch` gating, not covered by a written test yet.

## Out of scope

- Any change to the existing publish/community/org visibility model (Palette §6) — this feature reuses it as-is for the **id-only** case only; the data-bearing link deliberately sits outside it, see Rules.
- **Link expiry or revocation — explicitly ruled out by the requester** ("ce n'est absolument pas prévu"), not merely unaddressed.
- **Restricting who can open a data-bearing link — explicitly ruled out by design**, per the requester: these links are meant to be public/shareable to anyone, not a confidential access-control surface. Any future confidentiality/audience control would live in [Team spaces](team-spaces.md), not here — see Rules.
- Sharing a palette *into* a design-tool plugin via this same link mechanism — this spec covers the Web App only (see [Web App](../03-platform-bridges/web-app.md)).
- Compressing/shortening the data-bearing link — not implemented, and the requester confirmed the current uncompressed JSON is acceptable as-is (see Rules).

## Implementation notes

- **Web App**: no dedicated `/p/:id` route — the palette id is the path segment of `/palettes/:id`, optionally with `?data=<json>` (previously `/manage?id=<id>[&data=<json>]`, before the 2026-10 rename), resolved on mount by `resolvePaletteFromUrl` (`src/data/urlPalette.ts`), called from `src/pages/manage.tsx`. The visible URL is then normalized to `/palettes/<id>` (data stripped) by `useSyncPaletteUrl` (`src/ui/useSyncPaletteUrl.ts`) whenever `$palette.id` changes, and to `/palettes/local` when there's no active palette.
- **API**: the id-bearing case does **not** go through the documented `GET /v1/get-published-palette/:id` REST endpoint (see [`04-contracts/api-endpoints.md`](../04-contracts/api-endpoints.md)) — it queries Supabase directly (`webConfig.dbs.palettesDbViewName`, filtered by `palette_id`) from the client. Whether the API layer is meant to sit in front of this eventually is open.
- **Encoding**: no compression — raw `JSON.stringify`, see Rules.
- **Credits/plan**: generating or opening a share link does not go through the credits system in the code read so far (`copyShareLink`/`resolvePaletteFromUrl` don't reference `fees` or dispatch any credit-debit message) — tentatively confirms the original assumption, not yet cross-checked against the rest of the credits spec.
- **Access-denied UX**: confirmed by the requester as deliberately minimal for now — a lightweight message ("un léger toast") stating the palette isn't accessible, with no "request access" or "sign in as a different account" affordance. A more elaborate flow is acknowledged as a plausible future improvement, but nothing is planned — treat the current bare message as the intended v1 behavior, not a placeholder.

## Locales

| Text | Where | Tolgee key | Status |
| --- | --- | --- | --- |
| 2026-10-07 | URL shape updated to match the routing reorganization: `/manage?id=<id>[&data=…]` is now `/palettes/<id>[?data=…]` (id as path segment); no-palette fallback is `/palettes/local`. Old `/manage?id=…` links are not redirected and no longer resolve — see [Web App](../03-platform-bridges/web-app.md#routing--ssr). |
| "Copy link" / "Share" action label | Web App palette toolbar | — | Unread — no mockup yet, not checked against Tolgee |
| Not-found / access-denied state copy | Web App, opening an invalid/private id-bearing link | — | Unread — no mockup yet |

## See also

- [Web App](../03-platform-bridges/web-app.md) — the only platform this applies to
- [Palette](../01-domain-model/palette.md) §6 — the ownership/visibility model this reuses
- [Team spaces](team-spaces.md) — whether a team-space palette can be shared this way is open (see Open questions there)
- [Modals](modals.md) — the pre-existing Publication modal/flow, which this is an additional, lighter transport alongside, not a replacement for

## Open questions

- ~~Does an id-bearing link also carry embedded data as a fallback, or is it id-only?~~ — resolved: the explicit "Copy link" action always embeds both; an id-only link only ever arises from copying the (rewritten) address bar. There is no "fallback to embedded data after a blocked remote check" — embedded data, when present, is tried first and unconditionally.
- ~~No compression is implemented for the data-bearing payload — is this acceptable long-term?~~ — resolved: confirmed acceptable as-is by the requester, no compression planned. See Rules.
- ~~Link expiry/revocation~~ — resolved: explicitly ruled out, not planned. See Out of scope.
- ~~Can a team-space palette be shared via this same mechanism, requiring team membership to open?~~ — resolved: team/workspace scoping is deferred entirely to later (see [Team spaces](team-spaces.md)); for now, every data-bearing link is public by design regardless of any team context — see Rules.
- ~~Does generating a link for an unpublished palette silently create a lightweight hosted record, or is the id-bearing case limited to already-published palettes?~~ — resolved: neither — the id is assigned at palette **creation**, always, and simply carries through to publication. Sharing never mints a new record. See Rules.
- ~~Should a blocked link surface a "request access"/"sign in as a different account" affordance?~~ — resolved: no, the current minimal message is the intended v1 behavior, not a placeholder. See Implementation notes.

*(none remaining as of this update — see History)*

## History

_Ordered most recent to oldest._

| Date | Change |
| --- | --- |
| 2026-09-21 | Status changed from Draft to **Implemented** — every design open question is now answered and the mechanism is confirmed live; only a test-coverage gap remains (see Acceptance criteria), not a design gap. |
| 2026-09-21 | Requester answered all five design Open questions from the same-day rewrite: data-bearing links are confirmed deliberately non-confidential/public (Palette §6 visibility only governs the id-only path), the id is assigned at palette creation (never minted lazily for sharing), link expiry/revocation and team-space scoping are both explicitly deferred/ruled out for now, no compression is planned (confirmed against a real share link), and the current minimal access-denied message is the intended v1 UX. Rules, Out of scope, Implementation notes, and Open questions updated accordingly. |
| 2026-09-21 | Rewrote User flow, Rules, Acceptance criteria, and Implementation notes against the actual `web-ui-color-palette` implementation (`urlPalette.ts`, `shareLink.ts`, `createPaletteFromLink.ts`, `useSyncPaletteUrl.ts`, `manage.tsx`). Key corrections: no dedicated `/p/` route (uses `/manage?id=&data=`), embedded data is tried *before* the remote check rather than as a fallback after a block, no compression is implemented, and the address-bar URL is rewritten to id-only after a palette loads — resolving several previously-open questions and surfacing new ones (size cap, credits gating, hosted-record creation). |
| 2026-08-23 | Created as Draft, consolidated from the requester's description of URL-based palette sharing with an optional hosted-palette id requiring an existence/ownership/shared check. No implementation exists yet. |
