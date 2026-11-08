# Feature — Modals

- **Status**: Implemented
- **Package(s) concerned**: `ui-ui-color-palette`
- **UI module**: `src/ui/modules/modals/` (15 modals)
- **Related spec**: [User context](../01-domain-model/user-context.md), [Palette](../01-domain-model/palette.md) (§6, publication lifecycle)

## Problem

Most of the 15 modals work as designed and were confirmed intentional by the author, but a few carry known legacy debt: the Onboarding modal is dead (replaced by a third-party driver library), a publication-specific modal-context entry is unused, and one pricing-trigger type shows no card at all. This spec documents current modal behavior and flags what's confirmed legacy vs. what's confirmed intentional.

## User flow

### Publishing a palette

1. From the palette editor's toolbar, the user opens the Publication modal (only available when the current palette is eligible).
2. If signed out, they see a sign-in wall with a single CTA.
3. If signed in, they see the palette's publish status (one of 9 states, e.g. not yet published / local changes ready to push / remote is newer / up to date / can be reverted) with the matching primary/secondary action (Publish, Sync, Revert, Detach, Unpublish).
4. They can toggle "Share" on/off before publishing or pushing changes.
5. They can star/unstar a published palette.

### Purchasing

1. The user opens Pricing (from various entry points) and sees the Week/Month/Year/Lifetime tabs turned on, with localized pricing, an "Ultimate" card (when turned on), and — depending on how Pricing was opened — an "Activate license" card or a custom-checkout card.
2. They start checkout, which opens externally or in an embedded payment iframe depending on the platform.
3. On success, they land on a "Welcome to Pro" confirmation.

### Activating a license

1. The user enters a license key and instance name, or reviews/validates/unlinks an existing one.
2. On success, they land on the same "Welcome to Pro" confirmation as a purchase.

### Everything else

- **Try Pro**: start a trial, or skip straight to purchase.
- **Announcements**: a CMS-driven carousel the user pages through, dismissed once seen.
- **About / Report / Store / Chat / Feedback**: informational or utility dialogs, no publish/payment logic.
- **Preferences**: opens straight into the language/deep-sync module (see `preferences.md`).
- **Notification banner**: a toast confirming success/failure of an action taken elsewhere, not something the user opens deliberately.

## Rules

- Publication's self/community/org visibility is **not** a publish-time choice — it's computed remotely (creator comparison for self, the "shared" flag for community, an org admin's management for org). The publisher only ever sees a single "Share" toggle.
- Pricing's checkout opens in an external browser on some platforms and as an embedded iframe on others (Penpot, Framer) — determined by which platforms allow embedding a third-party payment iframe, not by platform capability in general.
- Each plan can be turned off with its feature: `PRICING_PRO_WEEK`, `PRICING_PRO_MONTH`, `PRICING_PRO_YEAR`, `PRICING_PRO_LIFETIME` (tabs) and `PRICING_ULTIMATE` (the /ultimate card). Tabs keep their order and only list the plans turned on; with a single /pro plan left, the tabs are hidden; the first plan turned on is selected by default. Plan cards carry no static tag: only a running offer adds one (`−N%`).
- **Special offers** are calendar events (Black Friday, Christmas…), managed as discounts in Polar. A Polar discount is treated as a commercial event when it has **both a start date and an end date**: that is the whole rule. Undated discounts are never shown — they are partner or support codes tied to a context, not to the calendar (e.g. TLGUIDE20).
  - *Why dates and not a tag:* the first idea was to tag event discounts with a `campaign` metadata. Polar's dashboard cannot set metadata on a discount (only the API or CLI can), so a tag would make every event a developer task. Dates are set in the dashboard, and a time-boxed discount is a commercial event by nature.
  - *Exception:* a dated discount can still be kept off the pricing with the metadata `campaign = private`. It needs the API or CLI and is meant to stay rare.
  - *Dates:* the end is exclusive — "Ends at Nov 10" in the dashboard stops the offer at the very start of Nov 10 (dashboard dates are midnight, Paris time). For an offer running through Nov 30, set "Ends at Dec 1".
- The plugin lists the Polar discounts once at startup (fire-and-forget, 3 s timeout, only when Polar is enabled) and keeps the dated percentage ones that are not ended and not sold out, for the plan products they are restricted to (every plan product when unrestricted). At render time, Pricing keeps for each plan the offer whose window contains now (`startsAt <= now < endsAt`); if several match, the one with the latest start wins. The website applies the same rules server-side.
- An active offer discounts the Polar price shown in the card subtitle (`round(amount * (1 - percent / 100))`, formatted like regular prices), preceded by the Polar price and an arrow, e.g. "€75.99 → €37.99 per user/year" (the subtitle is plain text, so no strikethrough). The card tag shows `−N%`, not localized, and the plan tab is marked as new — "new" only flags plans with a running offer. The card says nothing about how long the discount lasts: Polar's checkout tells the customer. Checkout receives the discount id (and code, if any). Outside Pricing, the code of the running offer (its name when it has no code) is shown as a tag right next to the Go Pro / Try Pro button, for users without the Pro plan; when several offers run on the plans turned on, the biggest discount wins. If the Polar price is unavailable, the hardcoded fallback price stays, without the arrow, and the `−N%` tag still shows.
- Offers never block Pricing or checkout: any failure (network, timeout, missing `discounts:read` scope, Polar disabled, no dated discount) means no offer and nominal pricing.
- Completing a purchase while a trial is active cancels the trial (it moves to `SUSPENDED`) — trial and paid plan are mutually exclusive from that point.
- License unlinking is local-only (no network round-trip) if the last validation check errored; otherwise it also deactivates the key remotely.
- Onboarding is dead: a separate third-party driver library handles onboarding today, not this modal.

## Acceptance criteria

- [ ] Given a palette the current user owns, when its remote copy is newer, then the modal offers Sync and Detach, not Publish.
- [ ] Given a palette the current user does not own, when viewing a version that can be pulled, then the modal resolves it toward an "up to date" state rather than offering ownership-only actions.
- [ ] Given the live pricing call fails or returns empty, when Pricing renders, then the hardcoded fallback prices are shown instead of a broken or empty card.
- [ ] Given a dated Polar discount is currently running for a plan, when Pricing renders, then that plan's card shows the Polar price, an arrow and the discounted price and a `−N%` tag, its tab is marked as new, the discount code shows next to the Go Pro button, and checkout receives the discount id.
- [ ] Given a Polar discount without a start or end date, or with the metadata `campaign = private`, or a fixed-amount one, or one that reached its redemption limit, when Pricing renders, then no offer is shown for it.
- [ ] Given an offer is upcoming (not started) or expired (ended), when Pricing renders, then no offer is shown and the nominal price is used.
- [ ] Given listing the Polar discounts fails or times out, when Pricing renders, then nominal pricing is shown exactly as without offers and Pricing/checkout are not delayed.
- [ ] Given an offer is active but the live Polar price is unavailable, when Pricing renders, then the hardcoded fallback price is shown, without the arrow, with the `−N%` tag.
- [ ] Given Pricing was opened via the "Activate license" trigger, when it renders, then the Activate card is shown; given it was opened via the custom-checkout trigger, then that card is shown instead; given the third trigger type, then no extra card is shown. *(confirmed legacy for the third case — target behavior not yet defined)*
- [ ] Given a purchase or license activation succeeds, when it completes, then the "Welcome to Pro" confirmation is shown, and any active trial is cancelled (moved to `SUSPENDED`), unless it was never used.
- [ ] Given the license's last validation check errored, when the user unlinks it, then no network call is made — the key is only cleared locally.
- [ ] Given the Onboarding modal, when its legacy status-check path fires, then it should not attempt to open it — onboarding is fully owned by the third-party driver library instead. *(currently the check still posts the message even though the handler is disconnected — worth cleaning up the dead path, not just leaving the handler disconnected)*

## Out of scope

- Reactivating or redesigning the Onboarding modal — it's confirmed dead, replaced by a third-party library; this spec doesn't propose bringing it back.
- Defining a target behavior for the third pricing-trigger type showing no card — confirmed legacy, deliberately left unspecified here.
- Removing the unused publication-specific entry in the generic modal-context enum — confirmed legacy cleanup, not scoped in this pass.

## Implementation notes

Reuses shared app-level types (`Editor`, `PlanStatus`, `Service`, translation/config context). Publication, Pricing, License, and the Announcements/Onboarding pair each have their own local state shape; the rest are effectively stateless containers. Not duplicated here — see `../04-contracts/events-messages.md` for the analytics payload shapes.

- **Outgoing channel**: modals only ever talk to the host sandbox through the same generic messaging channel every other module uses (toast display, Try Pro/license flows, checkout, item storage, "open in browser").
- **Central dispatch**: which modal is open is decided one level up, not inside the modal components themselves.
- **Bridges**: trial status and license status are resolved by dedicated startup checks; announcements/onboarding versioning is resolved by its own check.
- **Stores**: none of the 15 modals reads the app-wide consent or history state directly.
- **Analytics**: publication actions, pricing views/purchases, sign-in, and announcement/onboarding navigation are tracked; purchase and trial-enablement events are tracked one level above the modals themselves. All gated by analytics being enabled and user consent.
- **Platforms**: Pricing's checkout embed-vs-external-browser split is the only platform-specific branch identified across the 15 modals; publication is explicitly enabled on Penpot via the shared feature-flag config, not from inside the modal.

## See also

- [Palette](../01-domain-model/palette.md) §6 — the publication status machine summarized here
- [User context](../01-domain-model/user-context.md) — plan/trial types and consent backing Pricing, License, Try Pro
- [Palettes](palettes.md) — local-side counterpart of the Publication modal
- [Preferences](preferences.md) — the module the Preferences modal simply contains
- [Bridge messages & analytics events](../04-contracts/events-messages.md) — `PublicationEvent`, `PricingEvent`, `TourEvent`, `LanguageEvent` payloads

## History

| Date | Change |
| --- | --- |
| 2026-07-27 | Created — as-is consolidation from an agent's code reading |
| 2026-07-30 | Rewritten at a functional level (behavior/edge cases instead of file/line references), internal links added |
| 2026-08-03 | Reformatted to the Problem/User flow/Rules/Acceptance criteria template; the previously open author Q&A is now folded into Rules/Out of scope since every point was resolved |
| 2026-10-29 | Plan, gating and credits content removed; plans live in the docs (free-and-pro) |
| 2026-10-29 | Confirmed by the product owner: a purchase during an active trial cancels the trial (rule and acceptance criterion restored) |
