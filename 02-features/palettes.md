# Feature — Palettes (local & remote management)

- **Status**: Draft (as-is consolidation — no behavior change proposed at this stage)
- **Package(s) concerned**: `ui-ui-color-palette`
- **UI module**: `src/ui/modules/palettes`
- **Related spec**: [Palette](../01-domain-model/palette.md) (§6), [Bridge catalog](../03-platform-bridges/bridge-actions.md)

## 1. Context

This spec documents how **local** palette management works today — listing, creation, duplication, opening, deletion — as exposed by the two "local palettes" sub-tabs (one scoped to the file, one to the current page). No proposed evolution at this stage; several unexplained behavioral divergences are flagged in §6 to settle before touching this module.

## 2. Current behavior (as-is)

The file-scoped and page-scoped views are two **near-twin** implementations, sharing the same props from their parent (list status, the palette list itself, source colors, create/explore callbacks) and largely the same behavior.

**List**: sorted by last-opened date, most recent first. No search, no filter, no drag & drop. Each row shows the palette's name (a placeholder if unnamed), a "Published" badge when applicable, its preset, a short summary, and a live color preview of its active theme.

**Row actions**: a menu (Duplicate, Delete) plus a direct Open/View button.
- **Duplicate** creates a copy under a new id, blocked once the local-palette quota is reached (in addition to being plan-gated).
- **Delete** asks for confirmation first, and is **never** blocked by the quota — you can always delete your way back under the limit.
- **Open** and **View** currently send the exact same action with no distinction between them on the UI side — what should differentiate a read-only "view" from an editable "open" isn't implemented yet (see §6).
- A loading spinner on the acted-upon row is reset globally whenever the list changes length, not scoped to the specific row that triggered it — a fast duplicate-then-delete elsewhere in the list could theoretically clear the wrong spinner (see §6).

**Empty state**: a message with "Explore palettes" and "Create a palette" calls to action (the latter blocked if the quota is reached), plus a live preview of the currently selected source colors.

**Quota banner**: shown once the local-palette limit is reached, except in dev-oriented editors, with the numeric limit and a trial/upgrade prompt.

**Underlying operations**:
- Creating a palette builds a full configuration with one default theme.
- Duplicating re-reads the source entry, renames it as a copy, and regenerates its id/dates/status.
- Adding a palette pulled from the published-palette browser refuses if that id already exists locally.
- Listing rebuilds the local list by scanning everything stored locally that looks like a palette — **it isn't actually scoped by page or file** on the shared UI side, despite the sub-tab naming (see below, resolved).
- Opening a palette updates its "last opened" timestamp before loading it.
- Deleting a palette is a straight removal, no confirmation replay, no soft-delete.

## 3. Proposal (to-be)

None — consolidation spec.

## 4. Data model

Both views share the same props/state shape: list status (loading/loaded/empty), the palette list, source colors, create/explore callbacks; local state tracks the delete-confirmation dialog, the targeted palette, and a per-row loading flag. Each exposes the same set of plan-gated features (7 gates).

## 5. Impact

- **Bridges**: row actions route through the shared plugin-message dispatcher to the matching bridge (open/view → jump to palette, duplicate → create-from-duplication, delete → delete), each followed by a list refresh. Creating a brand-new palette or pulling one from the published browser is triggered one level up, not from these two views themselves.
- **Stores**: a shared "local palette count" value is written elsewhere and read by *other* modules (Explore, AI generation, color wheel, image import) for their own quota checks — these two views read the list length directly from their own props instead.
- **Credits**: duplicating a palette optimistically debits credits client-side, before the sandbox-side duplication is confirmed to have succeeded — with **no visible rollback if the duplication actually fails**. Open/View/Delete don't consume credits. Confirmed by the author as a real inconsistency to review.
- **Analytics**: duplicating, deleting, and opening/viewing a local palette have **no identifiable tracked event** today. Creating a palette and interacting with published palettes are tracked one level above this module.
- **Platforms**: no platform-specific branching inside these two views beyond hiding the quota banner in dev-oriented editors.

## 6. Open questions

- ~~The "current page" list has no notion of page/file despite its name~~ — **Answered (author)**: intended, not a bug. Scope actually differs by platform: Figma and Penpot scope at the page level, Framer and Sketch at the document level (Framer specifically reads from user-level storage, not document storage). This differentiation happens on the host/platform side, not in the shared UI code read here — to verify against each host repo if a future development touches this scoping.
- The file-scoped view requires both "create" and "browse community" to be available before showing the Explore CTA in the empty state, while the page-scoped view only requires "browse community" — intentional divergence between the two near-twin views, or drift?
- The delete-confirmation dialog falls back to a placeholder name in one view but not the other when the palette has no name — same question.
- Open and View send an identical action today: what should actually differentiate them (presumably read-only vs. editable), since nothing currently does?
- ~~No credit rollback if duplication fails~~ — **Answered (author)**: confirmed a real inconsistency, to review — add a rollback if the duplication bridge call fails.
- No search/filter/sort beyond implicit last-opened order — a deliberate limitation given the small quota (~3 palettes), or a gap for higher-tier plans?
- Delete is never blocked by the quota while Duplicate/Create are — confirmed intentional (lets you delete your way back under the limit)?
- A list-wide loading-spinner reset (not scoped per row) — could a delete on one row incorrectly clear a concurrent duplicate's spinner on another row?

## See also

- [Palette](../01-domain-model/palette.md) §6 — local vs. published lifecycle, the bridges named above
- [Bridge catalog](../03-platform-bridges/bridge-actions.md) — creation/read/deletion bridges behind these actions
- [Modals](modals.md) — the Publication modal, the remote/published-palette counterpart of this local view

## 7. History

| Date | Change |
| --- | --- |
| 2026-07-27 | Created — as-is consolidation from an agent's code reading |
| 2026-07-30 | Rewritten at a functional level (behavior/edge cases instead of file/line references), internal links added |
