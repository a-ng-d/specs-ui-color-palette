# Feature — Actions (top bar & mode switching)

- **Status**: Draft (as-is behavior, with confirmed edge cases flagged — see Acceptance criteria)
- **Package(s) concerned**: `ui-ui-color-palette`
- **UI module**: `src/ui/modules/Actions.tsx`, `src/ui/subservices/OpenPalette.tsx`
- **Related spec**: [Palette](../01-domain-model/palette.md), [Colors](colors.md), [Themes](themes.md), [Scale](scale.md), [Settings](settings.md), [Export](export.md), [Inspect](inspect.md), [Palettes](palettes.md), [Modals](modals.md)

## Problem

Once a palette is open, two tightly-coupled pieces own everything "around" Edit/Inspect/Export: `OpenPalette.tsx` owns the current `Mode` and renders the matching screen (`EditPalette`/`InspectPalette`/`ExportPalette`, each documented in their own siblings — Colors/Themes/Scale/Settings live inside Edit, see Export and Inspect); `Actions.tsx` renders the top bar for all three (back button, palette name, publish chip, sync/document/export actions, and the mode switcher itself). Both work as intended, but the palette name can be edited from two different places, and the mode switcher has an edge case where it can select a mode that isn't actually rendered.

## User flow

1. Once a palette is open, a top bar renders differently per mode:
   - **Edit**: back button, inline-editable palette name (64-char limit), a "Published" chip when applicable, a view-format dropdown (Palette / Palette with properties / Sheet) when a document already exists for this palette, a document-generation menu (same three formats, plus "push updates" once a generated document is out of date), a sync menu (styles/variables/tokens), and the mode switcher.
   - **Inspect**: back button, read-only palette name, "Published" chip, mode switcher only — no sync/document/export actions.
   - **Export**: back button, read-only palette name, "Published" chip, a primary "Export as {format}" button, mode switcher.
2. A segmented control (the mode switcher) lets the user jump between Edit, Inspect, and Export at any time — each option only shown if that mode is plan/platform-active.
3. On narrow viewports (≤460px), the Edit-mode top bar collapses the view/document/sync actions into a single icon menu with nested groups, instead of separate dropdowns/menus.

## Rules

- The palette name is editable in **two places**: inline in Actions' Edit-mode top bar, and again in Settings' name field (see Settings) — both write to the same store key and send the same `UPDATE_PALETTE` message, so they stay in sync, but it's still two separate inputs a user could land on.
- Document generation (`GENERATE_PALETTE` / `GENERATE_PALETTE_WITH_PROPERTIES` / `GENERATE_SHEET`) and the "change view" dropdown expose the **same three target formats** through two different affordances: generating always creates a new instance of that format; changing view only appears once a document already exists for this palette (`document.id === id`) and is meant to switch which format is currently shown — they're not the same action, but they overlap in what they offer.
- A "Push updates" option appears in the document menu only when a previously generated document is stale (`document.updatedAt !== dates.updatedAt` for the same palette id) — computed on mount and on every update.
- Every credit-gated action here (document generation × 3, sync × 3) uses the same formula to decide if it's blocked: `isReached((creditsCount - fee) * -1 - 1)` in addition to the plan-based `isBlocked()` check — effectively "would this action put credits below zero," layered on top of the feature flag.
- `OpenPalette` computes its list of usable modes once, in the constructor, from which of `EDIT`/`INSPECT`/`EXPORT` are active, and defaults `state.mode` to the first of those — but falls back to the literal `'EDIT'` if none are active, rather than to an empty/blocked state. *(edge case — see Acceptance criteria)*
- `OpenPalette.features` also defines `BROWSE` and `PUBLICATION` flags that are never read anywhere in the file — dead code, or reserved for a future check.
- The mode switcher itself (`Actions.Modes`) is shared verbatim across all three top-bar variants (Deploy/Inspect/Export), driven by `props.mode` and a single `onChangeMode` callback owned by `OpenPalette`.

## Acceptance criteria

- [ ] Given no mode is active for the current plan/platform (a misconfiguration), when `OpenPalette` mounts, then it should not default to `'EDIT'` if `EDIT` itself isn't active — today it does, which would render nothing (the `Feature isActive` guard blocks `EditPalette`) instead of a defined empty state. *(confirmed edge case, fix pending)*
- [ ] Given a document exists for the current palette and it's stale, when the top bar renders, then "Push updates" appears in the document menu; when it's not stale, it doesn't.
- [ ] Given a credit-gated action (document generation or sync) would leave the user with negative credits, when they try to trigger it, then it's blocked with a trial/upgrade prompt, consistent with the shared `isReached` formula.
- [ ] Given the palette name is edited from either Actions' top bar or Settings' name field, when the change is confirmed, then both reflect the same value afterward (shared store key).

## Out of scope

- Removing the `BROWSE`/`PUBLICATION` dead feature flags from `OpenPalette` — flagged, not acted on here.
- Unifying "generate document" and "change view" into a single affordance — documented as an overlap, not a redesign.
- Any change to what Edit/Inspect/Export themselves contain — fully owned by their own specs (Colors/Themes/Scale/Settings/Export/Inspect).

## Implementation notes

`OpenPalette` is a thin router: it owns `state.mode`, exposes `setMode`/`setEditContext`/`setInspectContext` (used by the onboarding tour, see `startTour`), and renders whichever of `EditPalette`/`InspectPalette`/`ExportPalette` matches. `Actions` is stateless beyond a small `canUpdateDocument` derived flag and a tooltip-visibility flag; it receives `mode` and a grab-bag of optional callbacks (`onSyncLocalStyles/Variables/Tokens`, `onGenerateDocument`, `onChangeView`, `onExportPalette`, `onUnloadPalette`) from whichever mode currently renders it.

- **Stores**: reads `$palette` directly only for the name field's live value; every other value comes through props from the owning mode.
- **Bridges**: `UPDATE_PALETTE` (name edit), `SYNC_LOCAL_STYLES` / `SYNC_LOCAL_VARIABLES` / `SYNC_LOCAL_TOKENS`, `CREATE_DOCUMENT` / `UPDATE_DOCUMENT` (document generation and push-updates) — all fire from `EditPalette`'s handlers, passed down as callbacks; `Actions` itself never calls `sendPluginMessage` for these, only for the generic `GET_TRIAL`/`GET_PRO` upsell on any blocked action.
- **Analytics**: document generation and sync actions are tracked via `trackActionEvent` one level up, in `EditPalette` — not inside `Actions` itself.
- **Credits**: document generation (3 formats) and sync (3 targets) each have their own fee, debited optimistically client-side before the host confirms — same pattern as duplication in Palettes (no confirmed rollback-on-failure gap identified here, unlike Palettes' duplicate bug, but not verified either).
- **Platforms**: the ≤460px collapse to a single icon menu is the only responsive/platform-shaped branching found in this module.

## Open questions

*(none)*

## See also

- [Palettes](palettes.md) — the row-level "Open" action that leads into this top bar via `OpenPalette`
- [Export](export.md), [Inspect](inspect.md) — the two non-Edit modes this top bar also renders for
- [Settings](settings.md) — the second place the palette name can be edited
- [Modals](modals.md) — the Publication modal, reached from the "Published" chip / publish flow referenced here

## History

| Date | Change |
| --- | --- |
| 2026-08-04 | Created — as-is consolidation from reading `Actions.tsx` and `OpenPalette.tsx`, following up on a request to cover the app's mode-switching chrome |
