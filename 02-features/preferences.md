# Feature — Preferences (user preferences)

- **Status**: Draft (as-is consolidation — no behavior change proposed at this stage)
- **Package(s) concerned**: `ui-ui-color-palette`
- **UI module**: `src/ui/modules/preferences`
- **Related spec**: [User context](../01-domain-model/user-context.md)

## 1. Context

This spec documents two preference groups: interface language, and "deep sync" with the host editor's native styles/variables/tokens. Consolidation only, no proposed evolution.

## 2. Current behavior (as-is)

### Language

A dropdown of 7 supported languages (English, Spanish, French, Brazilian Portuguese, Simplified Chinese, Japanese, Korean). The current selection is always read live from the translation engine, not from local component state.

Changing language: switches the active translation, re-translates preset names and consent labels, notifies the host side to persist the choice, and tracks the change. Every language is available on every plan today — language choice isn't paywalled, unlike deep sync below.

Persistence happens on the host side (stored locally per install), re-applied on startup, defaulting to the product's configured default language if nothing was saved yet.

### Deep sync

Three independent on/off toggles — styles, variables, tokens — each persisted on the host side as soon as it's flipped.

The user-facing copy makes an explicit promise: *"When enabled, the palette will be synchronized with the local library, so the local items will be updated, but could be removed."* **Important**: today, flipping these toggles only persists a boolean — it doesn't trigger any actual synchronization. The real sync execution these toggles are meant to gate is currently a stub on the host side (it just logs and shows a toast). This is a **promised-but-not-yet-wired contract**, not a bug in the toggle itself (see §6).

"Local" here means the host editor's own native styles/variables/tokens (Figma styles/variables, Penpot tokens, etc.) — not this product's remote/published palettes.

## 3. Proposal (to-be)

None — consolidation spec.

## 4. Data model

Reuses the shared app-level context types (plan, editor, translation). Deep sync state is three independent booleans, one per artifact type (styles/variables/tokens).

## 5. Impact

- **Message chain**: a language change or a deep-sync toggle both dispatch through the same generic UI→host messaging channel, ending in a value written to host-local storage.
- **Init bridge**: a startup check initializes these preferences on first launch (deep sync off by default, language defaulting to the product's configured default) and republishes the full preference state back to the UI.
- **Analytics**: language changes are tracked (with the resolved language value). **Deep-sync toggles are not tracked at all** — only persisted.
- **Credits / plan gating**: the three deep-sync toggles are plan-gated (a paid perk), computed the same way credit-consuming actions are gated elsewhere — see the credits table in `../01-domain-model/user-context.md` (styles/variables/tokens sync each have their own cost entry). Language choice carries no such gating.
- **Platforms**: which deep-sync sub-toggles are shown depends on the host editor's actual capabilities — styles sync is available everywhere, variables sync only where the platform has a native variables concept, tokens sync only where the platform has a native token concept. Language has no platform restriction.

## 6. Open questions

- Is there any local component state for the selected language that's declared but never actually used, given the source of truth is read live from the translation engine? Not confirmed — don't treat any such state as meaningful until verified.
- Deep-sync toggles aren't tracked in analytics, unlike almost every other preference change — deliberate, or a gap to fill?
- The toggle copy promises actual synchronization ("local items will be updated, but could be removed") while the real sync execution is currently a stub. Worth documenting the **target** behavior (the promised contract) separately from the **currently implemented** one (preference stored, sync not wired) so this gap doesn't get missed.
- There's an adjacent automatic language-suggestion feature wired elsewhere in the app shell — should it be folded into this spec or documented separately?
- One language code uses a longer regional-variant format than the others (a Simplified Chinese identifier vs. the simple `xx-XX` pattern everywhere else) — confirm this is intentional (a standard locale identifier), not a typo.

## See also

- [User context](../01-domain-model/user-context.md) — plan gating behind deep sync, credit costs
- [Modals](modals.md) — the Preferences modal is a thin container for this module
- [Platform bridges](../03-platform-bridges/bridge-actions.md) — where deep sync would eventually plug in (styles/variables/tokens sync are documented per platform)

## 7. History

| Date | Change |
| --- | --- |
| 2026-07-27 | Created — as-is consolidation from an agent's code reading |
| 2026-07-30 | Rewritten at a functional level (behavior/edge cases instead of file/line references), internal links added |
