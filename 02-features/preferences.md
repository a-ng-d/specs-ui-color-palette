# Feature — Preferences (user preferences)

- **Status**: Implemented
- **Package(s) concerned**: `ui-ui-color-palette`
- **UI module**: `src/ui/modules/preferences`
- **Related spec**: [User context](../01-domain-model/user-context.md)

## Problem

Deep-sync preferences promise the user that enabling them will synchronize (and potentially remove) local items — but flipping the toggle today only stores a boolean; the actual sync it's meant to gate isn't wired up yet. This spec documents current preference behavior and flags that gap.

## User flow

1. The user picks an interface language from a dropdown of 7 supported languages — the choice applies immediately (retranslating presets and consent labels) and is available on every plan.
2. The user flips one or more "deep sync" toggles (styles, variables, tokens) — each is a paid feature, gated per plan, and only shown for artifact types the current host platform actually supports (e.g. variables sync only where the platform has a native variables concept).

## Rules

- Language choice carries no plan gating — free on every plan.
- Deep-sync toggles are plan-gated (a paid perk), using the same credit-based gating mechanism as other paid actions.
- Which deep-sync sub-toggles are shown depends on the host platform's actual capabilities: styles sync is available everywhere, variables sync only where the platform has a native variables concept, tokens sync only where it has a native tokens concept.
- Flipping a deep-sync toggle only persists the boolean — it does not trigger any synchronization today.
- Language defaults to the product's configured default if nothing was saved yet; deep-sync toggles default to off.
- The source of truth for the selected language is the user's saved preference; if none was saved yet, it falls back to the product's configured default (`en-US`).
- The 7 supported languages are `en-US`, `fr-FR`, `es-ES`, `pt-BR`, `ja-JP`, `ko-KR`, and `zh-Hans-CN` — the last one is intentionally longer (language-script-region) since Chinese needs to specify a script (`Hans` = simplified) in addition to the region; it's a valid BCP 47 identifier, not a typo.

## Acceptance criteria

- [ ] Given the language dropdown, when the user selects a language, then the interface, presets, and consent labels retranslate immediately, and the choice persists across sessions.
- [ ] Given any plan, when the user opens the language dropdown, then no option is paywalled.
- [ ] Given a host platform with no native variables concept, when the user opens deep-sync preferences, then the variables-sync toggle is not shown.
- [ ] Given a deep-sync toggle is flipped on, when the change is confirmed, then it is persisted, **and** the corresponding local library items are actually synchronized. *(currently only the boolean is persisted — the real sync is unwired; confirmed gap, fix pending)*
- [ ] Given a deep-sync toggle is flipped, when the change happens, then it is tracked in analytics the same way the language change is. *(currently not tracked at all — confirmed gap)*

## Out of scope

- Implementing the actual deep-sync execution (styles/variables/tokens synchronization) — this spec documents that it's promised but not built; building it is separate work.
- The automatic language-suggestion feature — it exists adjacent to this module but isn't covered here (kept as a separate concern). It surfaces as a permanent banner-style message, which the user can either dismiss (permanently) or confirm (permanently).

## Implementation notes

Reuses the shared app-level context types (plan, editor, translation). Deep-sync state is three independent booleans, one per artifact type (styles/variables/tokens).

- **Message chain**: a language change or a deep-sync toggle both dispatch through the same generic UI→host messaging channel, ending in a value written to host-local storage.
- **Init bridge**: a startup check initializes these preferences on first launch (deep sync off by default, language defaulting to the product's configured default) and republishes the full preference state back to the UI.
- **Analytics**: language changes are tracked (with the resolved language value). Deep-sync toggles are not tracked at all — only persisted.
- **Credits / plan gating**: the three deep-sync toggles are plan-gated, computed the same way credit-consuming actions are gated elsewhere — see the credits table in `../01-domain-model/user-context.md` (styles/variables/tokens sync each have their own cost entry). Language choice carries no such gating.
- **Platforms**: see Rules above for which sub-toggles show per platform.

## Open questions

*(none — see resolutions in History)*

## See also

- [User context](../01-domain-model/user-context.md) — plan gating behind deep sync, credit costs
- [Modals](modals.md) — the Preferences modal is a thin container for this module
- [Platform bridges](../03-platform-bridges/bridge-actions.md) — where deep sync would eventually plug in (styles/variables/tokens sync are documented per platform)

## History

| Date | Change |
| --- | --- |
| 2026-07-27 | Created — as-is consolidation from an agent's code reading |
| 2026-07-30 | Rewritten at a functional level (behavior/edge cases instead of file/line references), internal links added |
| 2026-08-03 | Reformatted to the Problem/User flow/Rules/Acceptance criteria template |
| 2026-08-04 | Resolved all three open questions: language source of truth (saved user preference, falling back to `en-US`), the language-suggestion banner stays out of scope (permanent, dismissible/confirmable forever), and `zh-Hans-CN` confirmed as an intentional BCP 47 identifier, not a typo |
