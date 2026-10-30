# Feature — Preferences (user preferences)

- **Status**: Implemented
- **Package(s) concerned**: `ui-ui-color-palette`
- **UI module**: `src/ui/modules/preferences`
- **Related spec**: [User context](../01-domain-model/user-context.md)

## Problem

Deep-sync preferences change what the next synchronization does with local items that are no longer in the palette: kept (off, the default) or deleted (on). Flipping a toggle stores the choice; the sync reads it. This spec documents current preference behavior.

## User flow

1. The user picks an interface language from a dropdown of 7 supported languages — the choice applies immediately (retranslating presets and consent labels).
2. The user flips one or more "deep sync" toggles (styles, variables, tokens) — each only shown for artifact types the current host platform actually supports (e.g. variables sync only where the platform has a native variables concept).

## Rules

- Which deep-sync sub-toggles are shown depends on the host platform's actual capabilities: styles sync is available everywhere, variables sync only where the platform has a native variables concept, tokens sync only where it has a native tokens concept.
- Flipping a deep-sync toggle persists the boolean; it does not run a synchronization by itself. The next sync of that artifact type reads it: when on, every item of the synced group, collection or token sets that no longer matches a shade of the palette is deleted, **without exception** — items the user added by hand to the same group, collection or sets included. When off, those items are kept. *(confirmed by the product owner, 2026-10-30)*
- Language defaults to the product's configured default if nothing was saved yet; deep-sync toggles default to off.
- The source of truth for the selected language is the user's saved preference; if none was saved yet, it falls back to the product's configured default (`en-US`).
- The 7 supported languages are `en-US`, `fr-FR`, `es-ES`, `pt-BR`, `ja-JP`, `ko-KR`, and `zh-Hans-CN` — the last one is intentionally longer (language-script-region) since Chinese needs to specify a script (`Hans` = simplified) in addition to the region; it's a valid BCP 47 identifier, not a typo.

## Acceptance criteria

- [ ] Given the language dropdown, when the user selects a language, then the interface, presets, and consent labels retranslate immediately, and the choice persists across sessions.
- [ ] Given a host platform with no native variables concept, when the user opens deep-sync preferences, then the variables-sync toggle is not shown.
- [ ] Given a deep-sync toggle is on, when the user syncs that artifact type, then every item of the synced group, collection or token sets that no longer matches a shade of the palette is deleted, hand-added items included; given it is off, those items are kept.
- [ ] Given a deep-sync toggle is flipped, when the change happens, then it is tracked in analytics the same way the language change is. *(currently not tracked at all — confirmed gap)*

## Out of scope

- The automatic language-suggestion feature — it exists adjacent to this module but isn't covered here (kept as a separate concern). It surfaces as a permanent banner-style message, which the user can either dismiss (permanently) or confirm (permanently).

## Implementation notes

Reuses the shared app-level context types (editor, translation). Deep-sync state is three independent booleans, one per artifact type (styles/variables/tokens).

- **Message chain**: a language change or a deep-sync toggle both dispatch through the same generic UI→host messaging channel, ending in a value written to host-local storage.
- **Init bridge**: a startup check initializes these preferences on first launch (deep sync off by default, language defaulting to the product's configured default) and republishes the full preference state back to the UI.
- **Analytics**: language changes are tracked (with the resolved language value). Deep-sync toggles are not tracked at all — only persisted.
- **Platforms**: see Rules above for which sub-toggles show per platform.

## Open questions

*(none — see resolutions in History)*

## See also

- [User context](../01-domain-model/user-context.md) — the user-context types this module reads
- [Modals](modals.md) — the Preferences modal is a thin container for this module
- [Platform bridges](../03-platform-bridges/bridge-actions.md) — where the styles, variables and tokens syncs that read the deep-sync toggles are documented per platform

## History

| Date | Change |
| --- | --- |
| 2026-07-27 | Created — as-is consolidation from an agent's code reading |
| 2026-07-30 | Rewritten at a functional level (behavior/edge cases instead of file/line references), internal links added |
| 2026-08-03 | Reformatted to the Problem/User flow/Rules/Acceptance criteria template |
| 2026-08-04 | Resolved all three open questions: language source of truth (saved user preference, falling back to `en-US`), the language-suggestion banner stays out of scope (permanent, dismissible/confirmable forever), and `zh-Hans-CN` confirmed as an intentional BCP 47 identifier, not a typo |
| 2026-10-29 | Plan, gating and credits content removed; plans live in the docs (free-and-pro) |
| 2026-10-30 | Confirmed by the product owner: deep sync works — the next sync reads the toggle and, when on, deletes every item no longer in the palette, without exception. The "unwired" gap, its acceptance criterion and the out-of-scope item are removed |
