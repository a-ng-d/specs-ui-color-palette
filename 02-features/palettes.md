# Feature — Palettes (local & remote management)

- **Status**: Draft (as-is behavior, with confirmed fixes pending — see Acceptance criteria)
- **Package(s) concerned**: `ui-ui-color-palette`
- **UI module**: `src/ui/modules/palettes`
- **Related spec**: [Palette](../01-domain-model/palette.md) (§6), [Bridge catalog](../03-platform-bridges/bridge-actions.md)

## Problem

Local palette management (list, create, duplicate, open, delete) is implemented as two near-twin views (file-scoped and page-scoped) that mostly work, but carry a few confirmed inconsistencies that need fixing: a credit debit with no rollback on a failed duplication, and unexplained divergences between the two twin views (empty-state CTA gating, delete-dialog fallback) that were never reconciled after the two views drifted apart.

## User flow

1. The user opens either the file-scoped or page-scoped local-palettes tab — both list the same local palettes, sorted by last-opened date, most recent first.
2. Each row shows the palette's name (a placeholder if unnamed), a "Published" badge when applicable, its preset, a short summary, and a live color preview of its active theme.
3. From a row, the user can:
   - **Open** or **View** it — intended to differ (Open enters edit mode, View only allows inspecting and exporting), but both currently do exactly the same thing (jump into the palette).
   - **Duplicate** it — creates a copy under a new id, with a loading spinner on that row while it processes.
   - **Delete** it — a confirmation dialog appears first, showing the palette's name.
4. If the local-palette quota is reached, a banner appears (except in dev-oriented editors) showing the limit and a trial/upgrade prompt; Create and Duplicate become blocked, Delete never does.
5. From the empty state, the user can jump to "Explore palettes" (published-palette browser) or "Create a palette".

## Rules

- No search, filter, or manual sort — only the implicit last-opened ordering.
- **Duplicate** is blocked once the local-palette quota is reached, in addition to being plan-gated.
- **Delete** is never blocked by the quota — a user can always delete their way back under the limit.
- **Open** and **View** are supposed to differ — Open opens the palette in edit mode (equivalent to Edit), View only allows inspecting and exporting it — but currently both trigger the identical action (jump to the palette).
- A palette pulled in from the published browser refuses to be added if that id already exists locally.
- Listing is **not actually scoped by page or file** in the shared UI code, despite the sub-tab naming — this is intentional: scoping happens on the host/platform side. Figma and Penpot scope at the page level, Framer and Sketch at the document level (Framer specifically reads from user-level storage, not document storage).
- The per-row loading spinner resets globally whenever the list's length changes, not scoped to the row that triggered it — harmless in practice, since the interface is blocked for the duration of the action, so no two rows can have a concurrent spinner to mix up.

## Acceptance criteria

- [ ] Given the local-palette list, when it renders, then rows are sorted by most-recently-opened first, with no search/filter/manual-sort control shown.
- [ ] Given a user duplicates a palette and the duplication succeeds, when the sandbox confirms success, then the row shows the new palette and the loading spinner clears.
- [ ] Given a user duplicates a palette and the duplication fails, when the failure is confirmed, then any credit debited for the action is rolled back. *(currently not the case — confirmed bug, fix pending)*
- [ ] Given the local-palette quota is reached, when the user tries to create or duplicate a palette, then the action is blocked with a trial/upgrade prompt.
- [ ] Given the local-palette quota is reached, when the user deletes a palette, then the action is never blocked by the quota.
- [ ] Given a palette has no name, when the delete-confirmation dialog opens, then it shows a translated placeholder instead of a blank name, consistently across both the file-scoped and page-scoped views. *(currently only true in one of the two — confirmed divergence, fix pending)*
- [ ] Given the empty state, when "browse community" is available, then the "Explore palettes" CTA is shown consistently across both the file-scoped and page-scoped views, regardless of whether "create" is also available. *(currently gated differently between the two views — confirmed divergence, fix pending)*

## Out of scope

- Search, filtering, or manual reordering of the local-palette list.
- Making Open and View behave differently in practice — the distinction is now defined (Open = edit mode, View = inspect/export only) but not yet implemented.
- Per-row-scoped loading state (today it's list-wide).
- Any change to how page/file scoping works on the host/platform side — that's owned by each platform host repo, not this shared module.

## Implementation notes

Both views share the same props/state shape: list status (loading/loaded/empty), the palette list, source colors, create/explore callbacks; local state tracks the delete-confirmation dialog, the targeted palette, and a per-row loading flag. Each exposes the same set of 7 plan-gated features.

- **Bridges**: row actions route through the shared plugin-message dispatcher to the matching bridge (open/view → jump to palette, duplicate → create-from-duplication, delete → delete), each followed by a list refresh. Creating a brand-new palette or pulling one from the published browser is triggered one level up, not from these two views themselves.
- **Stores**: a shared "local palette count" value is written elsewhere and read by *other* modules (Explore, AI generation, color wheel, image import) for their own quota checks — these two views read the list length directly from their own props instead.
- **Credits**: duplicating a palette optimistically debits credits client-side, before the sandbox-side duplication is confirmed to have succeeded — see the rollback fix above.
- **Analytics**: duplicating, deleting, and opening/viewing a local palette have no identifiable tracked event today. Creating a palette and interacting with published palettes are tracked one level above this module.
- **Platforms**: no platform-specific branching inside these two views beyond hiding the quota banner in dev-oriented editors.

## See also

- [Palette](../01-domain-model/palette.md) §6 — local vs. published lifecycle, the bridges named above
- [Bridge catalog](../03-platform-bridges/bridge-actions.md) — creation/read/deletion bridges behind these actions
- [Modals](modals.md) — the Publication modal, the remote/published-palette counterpart of this local view

## History

| Date | Change |
| --- | --- |
| 2026-07-27 | Created — as-is consolidation from an agent's code reading |
| 2026-07-30 | Rewritten at a functional level (behavior/edge cases instead of file/line references), internal links added |
| 2026-08-03 | Reformatted to the Problem/User flow/Rules/Acceptance criteria template |
| 2026-08-04 | Resolved the Open vs. View open question: Open enters edit mode, View allows inspection/export only (not yet implemented) |
| 2026-08-04 | Resolved the spinner-reset open question: harmless, since the interface blocks concurrent actions — no two rows can race; "Open questions" section removed |
