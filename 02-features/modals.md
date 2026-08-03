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
5. They can star/unstar a published palette (a paid action, prompting a trial/upgrade flow if blocked).

### Upgrading

1. The user opens Pricing (from various entry points) and sees Week/Month/Year/Lifetime tabs with localized pricing, an "Ultimate" card, and — depending on how Pricing was opened — an "Activate license" card or a custom-checkout card.
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
- Starring a palette is plan-gated; if blocked, it opens a trial/upgrade prompt instead of starring.
- Pricing's checkout opens in an external browser on some platforms and as an embedded iframe on others (Penpot, Framer) — determined by which platforms allow embedding a third-party payment iframe, not by platform capability in general.
- The "Best deal" tag on the Monthly pricing tier is static, not computed from actual pricing — intentional.
- License unlinking is local-only (no network round-trip) if the last validation check errored; otherwise it also deactivates the key remotely.
- Onboarding is dead: a separate third-party driver library handles onboarding today, not this modal.
- Completing a purchase while a trial is active suspends the trial — trial and paid plan become mutually exclusive from that point.

## Acceptance criteria

- [ ] Given a palette the current user owns, when its remote copy is newer, then the modal offers Sync and Detach, not Publish.
- [ ] Given a palette the current user does not own, when viewing a version that can be pulled, then the modal resolves it toward an "up to date" state rather than offering ownership-only actions.
- [ ] Given starring is plan-blocked, when the user clicks the star button, then a trial/upgrade prompt opens instead of the palette being starred.
- [ ] Given the live pricing call fails or returns empty, when Pricing renders, then the hardcoded fallback prices are shown instead of a broken or empty card.
- [ ] Given Pricing was opened via the "Activate license" trigger, when it renders, then the Activate card is shown; given it was opened via the custom-checkout trigger, then that card is shown instead; given the third trigger type, then no extra card is shown. *(confirmed legacy for the third case — target behavior not yet defined)*
- [ ] Given a purchase or license activation succeeds, when it completes, then the "Welcome to Pro" confirmation is shown, and any active trial is moved to a suspended state (unless it was never used).
- [ ] Given the license's last validation check errored, when the user unlinks it, then no network call is made — the key is only cleared locally.
- [ ] Given the Onboarding modal, when its legacy status-check path fires, then it should not attempt to open it — onboarding is fully owned by the third-party driver library instead. *(currently the check still posts the message even though the handler is disconnected — worth cleaning up the dead path, not just leaving the handler disconnected)*

## Out of scope

- Reactivating or redesigning the Onboarding modal — it's confirmed dead, replaced by a third-party library; this spec doesn't propose bringing it back.
- Defining a target behavior for the third pricing-trigger type showing no card — confirmed legacy, deliberately left unspecified here.
- Removing the unused publication-specific entry in the generic modal-context enum — confirmed legacy cleanup, not scoped in this pass.
- Adding credit consumption to any of the 15 modals — the credits system is currently inactive product-wide; this isn't the spec that reactivates it.

## Implementation notes

Reuses shared app-level types (`Editor`, `PlanStatus`, `Service`, translation/config context) plus a per-modal plan-gating flag. Publication, Pricing, License, and the Announcements/Onboarding pair each have their own local state shape; the rest are effectively stateless containers. Not duplicated here — see `../04-contracts/events-messages.md` for the analytics payload shapes.

- **Outgoing channel**: modals only ever talk to the host sandbox through the same generic messaging channel every other module uses (toast display, trial/pro/license flows, checkout, item storage, "open in browser").
- **Central dispatch**: which modal is open is decided one level up, not inside the modal components themselves.
- **Bridges**: trial status and license status are resolved by dedicated startup checks; announcements/onboarding versioning is resolved by its own check.
- **Stores**: none of the 15 modals reads the app-wide consent, credits, or history state directly — credit logic, where it applies, happens upstream of a modal being opened.
- **Analytics**: publication actions, pricing views/purchases, sign-in, and announcement/onboarding navigation are tracked; purchase and trial-enablement events are tracked one level above the modals themselves. All gated by analytics being enabled and user consent.
- **Platforms**: Pricing's checkout embed-vs-external-browser split is the only platform-specific branch identified across the 15 modals; publication is explicitly enabled on Penpot via the shared feature-flag config, not from inside the modal.
- **Credits**: no credit consumption identified in any of the 15 modals — see the credits note in `../01-domain-model/user-context.md` (currently inactive product-wide, but not to be treated as permanently out of scope).

## See also

- [Palette](../01-domain-model/palette.md) §6 — the publication status machine summarized here
- [User context](../01-domain-model/user-context.md) — plan/trial/credits/consent backing Pricing, License, Try Pro
- [Palettes](palettes.md) — local-side counterpart of the Publication modal
- [Preferences](preferences.md) — the module the Preferences modal simply contains
- [Bridge messages & analytics events](../04-contracts/events-messages.md) — `PublicationEvent`, `PricingEvent`, `TourEvent`, `LanguageEvent` payloads

## History

| Date | Change |
| --- | --- |
| 2026-07-27 | Created — as-is consolidation from an agent's code reading |
| 2026-07-30 | Rewritten at a functional level (behavior/edge cases instead of file/line references), internal links added |
| 2026-08-03 | Reformatted to the Problem/User flow/Rules/Acceptance criteria template; the previously open author Q&A is now folded into Rules/Out of scope since every point was resolved |
