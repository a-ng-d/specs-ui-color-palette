# Feature — Modals

- **Status**: Draft (as-is consolidation — no behavior change proposed at this stage)
- **Package(s) concerned**: `ui-ui-color-palette`
- **UI module**: `src/ui/modules/modals/` (15 modals)
- **Related spec**: [User context](../01-domain-model/user-context.md), [Palette](../01-domain-model/palette.md) (§6, publication lifecycle)

## 1. Context

This spec documents the 15 modals of the application: palette publishing, monetization (pricing/license), onboarding/announcements, and utility modals (about, report, store, preferences, chat, feedback, notifications). This is the largest module and the one carrying the most sensitive business logic (publishing, payment, licensing) — handle with care for any future development.

## 2. Current behavior (as-is)

### Publication

Not routed through the generic modal system like the others — it's mounted directly inside the palette editor, gated by whether the current palette is eligible for publishing, and opened from a dedicated toolbar button.

Two renders depending on whether the user is signed in:
- **Signed in**: title "Publish" if the current user owns the palette (or it has no owner yet), otherwise "Sync"; palette preview; name/preset/description; a status chip; the original creator's avatar shown only when viewing a palette published by someone else and its status allows re-syncing; a star (favorite) button except when the palette isn't published or the remote copy can't be found.
- **Signed out**: a sign-in wall with a single "Sign in" call to action.

A 9-state publication status machine is computed by comparing local timestamps against the remote copy and checking ownership. Each status maps to a primary/secondary action pair:

| Status | Action(s) |
| --- | --- |
| Not yet published | Publish |
| Local changes ready to push | Publish / Revert |
| Remote is newer (you own it) | Sync / Detach |
| Remote is newer (someone else owns it) | resolves to "up to date" |
| Published | Publish (disabled unless sharing changed) / Unpublish |
| Up to date | Detach only |
| Can be reverted | Revert / Detach |
| Remote copy not found | Detach only |
| Pending | actions disabled |

**Visibility**: from the publisher's side, sharing is a simple boolean toggle ("Share": yes/no), available whenever the palette isn't in a "remote is newer, not owned" state. The self/community/org distinction used elsewhere for *browsing* published palettes is **not** a publish-time choice — it's computed on the remote side (creator comparison for self, "shared" flag for community, an organization admin's management for org). See `../01-domain-model/palette.md` §6 for the full breakdown. This is intentional, not a missing feature.

Starring is plan-gated: if blocked, it opens a trial/upgrade prompt instead of starring.

### Pricing

Opens with 4 billing-period tabs (Week/Month/Year/Lifetime, defaulting to Week), one card per tier with live localized pricing and a hardcoded fallback price if the live pricing call fails or returns empty. The "Best deal" tag on the Monthly tier is static, not computed from actual pricing — confirmed intentional. An "Ultimate" card is always shown. An "Activate" (license key) card or a custom-checkout card appear only when the flow that opened Pricing specifically requested them; a third possible trigger type shows no extra card at all (confirmed legacy, not yet given a target behavior).

Checkout opens in an external browser on some platforms and as an embedded payment iframe on others (Penpot, Framer) — the split isn't about platform capability so much as which platforms allow embedding a third-party payment iframe. Purchase success is detected either via a message from the payment provider's page (embedded flow) or a manual "I've paid" button that polls the subscription status (external-browser flow).

### License

Lets a user activate a license key (with an instance name) if none is active yet, or view/validate/unlink an existing one. Unlinking is local-only if the last validation check errored (no network round-trip); otherwise it also deactivates the key remotely.

### Announcements / Onboarding

Both are CMS-driven carousels (image/title/tag/description) with "Next" / "Learn more" / "Got it" actions, each persisting a "last seen version" so they don't reappear. **Onboarding is currently unplugged**: it's legacy from a previous implementation — onboarding today is handled by a separate third-party driver library, not by this modal. Treat the Onboarding modal as dead in its current state.

### Utility modals

- **About**: plan badge (free/trial/pro/dev), author/license/repo links, third-party attributions.
- **Report**: name/email/message form sent via the crash-reporting tool's feedback API, with a session replay attached.
- **Store**: a single cross-sell card today, opens an external link.
- **Preferences**: a pure container for the language and deep-sync preferences (see `preferences.md`) — no logic of its own.
- **Try Pro**: primary action starts a trial, secondary skips straight to purchase.
- **Chat**: embeds a third-party live-chat widget, no business logic.
- **Welcome to Trial / Welcome to Pro**: confirmation dialogs shown right after a trial starts or a purchase/license activation succeeds.
- **Feedback**: an external survey embedded as an iframe, no local state.
- **Notification banner**: a toast, not a modal — fed by a generic "show this message" event most other modals also use to report success/failure.

## 3. Proposal (to-be)

None — consolidation spec.

## 4. Data model

Reuses shared app-level types (`Editor`, `PlanStatus`, `Service`, translation/config context) plus a per-modal plan-gating flag. Publication, Pricing, License, and the Announcements/Onboarding pair each have their own local state shape; the rest are effectively stateless containers. Not duplicated here — see `../04-contracts/events-messages.md` for the analytics payload shapes.

## 5. Impact

- **Outgoing channel**: modals only ever talk to the host sandbox through the same generic messaging channel every other module uses (toast display, trial/pro/license flows, checkout, item storage, "open in browser").
- **Central dispatch**: which modal is open is decided one level up, not inside the modal components themselves.
- **Bridges**: trial status and license status are resolved by dedicated startup checks; announcements/onboarding versioning is resolved by its own check.
- **Stores**: none of the 15 modals reads the app-wide consent, credits, or history state directly — credit logic, where it applies, happens upstream of a modal being opened. Displayed credit balance/renewal date exist in the shared context but aren't shown by any of these modals today.
- **Analytics**: publication actions, pricing views/purchases, sign-in, and announcement/onboarding navigation are tracked; purchase and trial-enablement events are tracked one level above the modals themselves. All gated by analytics being enabled and user consent.
- **Plan/trial impact on display**: the About badge reflects plan/trial state; starring is paywalled the same way trial/pro prompts are elsewhere; completing a purchase while a trial is active suspends the trial (trial and paid plan become mutually exclusive from that point).
- **Platforms**: Pricing's checkout embed-vs-external-browser split is the only platform-specific branch identified across the 15 modals; publication is explicitly enabled on Penpot via the shared feature-flag config, not from inside the modal.
- **Credits**: no credit consumption identified in any of the 15 modals — see the credits note in `../01-domain-model/user-context.md` (currently inactive product-wide, but not to be treated as permanently out of scope).

## 6. Open questions — all answered by the author (2026-07-27)

1. ~~Self/community/org visibility for Publication~~ — **Answered**: intentional. It's a computed *browsing* concept resolved remotely (creator comparison, shared flag, org admin management), not a publish-time toggle. See `../01-domain-model/palette.md` §6.
2. ~~Onboarding modal possibly dead~~ — **Answered**: confirmed legacy from the previous implementation; onboarding now runs through a third-party driver library instead. Not a priority to document further unless reactivated.
3. ~~A publication-specific entry in the generic modal-context enum is declared but never used~~ — **Answered**: confirmed legacy, to clean up later — not preparation for an ongoing unification.
4. ~~Hardcoded fallback prices~~ — **Answered**: intentional, kept on purpose.
5. ~~Static "Best deal" tag~~ — **Answered**: confirmed static, intended.
6. ~~The third pricing-trigger type shows no extra card~~ — **Answered**: confirmed legacy, to fix later (no target behavior specified yet).
7. ~~No credit consumption anywhere in the 15 modals~~ — **Answered**: the credits system is currently inactive product-wide, which explains the total absence of references here — it could become active again, so don't treat this as a permanent design decision.
8. ~~"About" is plan-gated even though its content isn't really plan-specific~~ — **Answered**: confirmed not necessarily relevant, can be fixed later. Not blocking.

## See also

- [Palette](../01-domain-model/palette.md) §6 — the publication status machine summarized here
- [User context](../01-domain-model/user-context.md) — plan/trial/credits/consent backing Pricing, License, Try Pro
- [Palettes](palettes.md) — local-side counterpart of the Publication modal
- [Preferences](preferences.md) — the module the Preferences modal simply contains
- [Bridge messages & analytics events](../04-contracts/events-messages.md) — `PublicationEvent`, `PricingEvent`, `TourEvent`, `LanguageEvent` payloads

## 7. History

| Date | Change |
| --- | --- |
| 2026-07-27 | Created — as-is consolidation from an agent's code reading |
| 2026-07-30 | Rewritten at a functional level (behavior/edge cases instead of file/line references), internal links added |
