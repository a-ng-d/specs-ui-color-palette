# Feature — Apply to selection (quick palette simulation)

- **Status**: Draft
- **Package(s) concerned**: `ui-ui-color-palette` (not implemented yet — new dropdown entry), per-platform host repos (`figma-ui-color-palette`, `penpot-ui-color-palette`, `sketch-ui-color-palette`, `framer-ui-color-palette`) for the actual canvas color-scan/replace/revert mechanics (not implemented yet, no shared bridge exists for this)
- **UI module**: none yet — proposed as a new entry in Edit mode's existing document/sync dropdown (`src/ui/modules/Actions.tsx`)
- **Related spec**: [Actions](actions.md) (the top-bar dropdown this plugs into), [Color System (feature)](color-system.md) (the separate, more technical semantic/template-based simulation this is explicitly kept independent from), [Preview](preview.md) (shade grid / theme switcher this dropdown lives beside)
- **Ticket**: Notion — "Simulation rapide de l'application de la palette sur un élément sélectionné" (`POR-645`)
- **Design**: Figma — `UICP Screens` file, node `4972:29275` ("Apply")

## Problem

Trying out a palette against an already-drawn design (a page, an illustration, a small component) today means either a full sync to styles/variables followed by manually reassigning every layer, or hand-picking replacement colors one by one — there's no fast way to just "try" a palette against existing artwork to see if it fits, before committing to it. The ticket frames this as a lightweight, non-semantic pre-check, explicitly distinct from a separate (not-yet-built) semantic simulation based on templates, which is more technical (role-aware) and out of scope here.

## User flow

1. The user selects a node on the canvas — a page, an illustration, or a small component containing multiple colors.
2. From Edit mode's top bar (see [Actions](actions.md)), the user opens the **renamed** "Apply to document…" dropdown — this is the existing "Synchronize…" menu (`actions.sync`), relabeled, with **"Apply to selection"** added as a new first entry above the existing "Sync with local styles" / "Sync with local variables" / "Sync with local tokens" actions.
3. Picking "Apply to selection" triggers the simulation: **every object in the selection, at any nesting depth, without exception**, that has a fill or a border/stroke set to a solid or gradient color is analyzed for its luminance — in short, every hex/RGB value found — entirely **client-side** (no API/server call). Image fills aren't affected (they're not a solid/gradient color value to begin with).
4. Each detected color is replaced by whichever shade of the palette's **active theme** has the **closest luminance** — purely by luminance, with no regard for role or context (no background/text/border/state distinction). For a gradient, each color stop is matched and replaced individually.
5. The result renders immediately on the canvas as a quick, non-committal preview.
6. The user can revert the selection back to its original colors at any time, via the platform's **native undo** (Cmd/Ctrl+Z) — no dedicated in-app "revert" action.

## Rules

- **Non-semantic by design**: luminance is the only criterion for matching a detected color to a palette shade — no role/context is considered. This is the ticket's explicit contrast with the separate, future semantic/template-based simulation (role-aware, more technical).
- **Client-side only**: luminance computation and the nearest-shade match happen without any server/API call — confirmed by the ticket. The palette's shades are already available locally (`PaletteDataShadeItem.hex` and other color-space fields, see `engine-ui-color-palette/src/types/data.types.ts`); luminance for both the detected canvas color and each candidate shade can be computed the same way contrast scoring already does elsewhere in the engine (`chroma(color).luminance()`), but the matching itself for this feature runs in the UI/bridge layer, not through an engine call.
- **Selection-scoped, non-destructive to the palette**: this only recolors existing nodes on the canvas — it never creates new nodes and never touches the palette itself (`base`/`themes` are untouched, consistent with how [Color System (feature)](color-system.md) frames its own semantic layer as additive/non-destructive).
- **Independent from the semantic/template-based simulation**: per the ticket, this quick mode must not interfere with or share implementation/state with that separate (not-yet-built) feature.
- Presumed theme-scoping: matches against the palette's **currently active theme's** shades only (consistent with how vision-simulation/text colors are theme-scoped elsewhere, see [Settings](settings.md)) — not confirmed against the ticket text itself, which doesn't mention multi-theme behavior; the mockup shows a theme-switcher dropdown ("Light ▾") nearby, in [Preview](preview.md)'s existing theme switcher, but whether this action reads it is unconfirmed (see Open questions).

## Acceptance criteria

- [ ] Given a selected element (page, illustration, or small component) containing multiple colors, when the user activates "Apply to selection," then every detected color is replaced by a palette color, chosen by closest luminance match.
- [ ] Given the quick simulation runs, then luminance computation and palette mapping happen entirely client-side, with no server/API call.
- [ ] Given the quick simulation's replacement, then no semantic role (background/text/border/state) is considered — luminance is the sole criterion.
- [ ] Given a quick-simulation result, then it is reversible — the user can return the selection to its original colors at any time.
- [ ] Given the semantic/template-based simulation (a separate, future feature), then this quick mode neither interferes with it nor shares its implementation or state.
- [ ] Given a palette with no semantic layer or unrelated feature active, when "Apply to selection" runs, then there is no regression to existing sync-to-styles/sync-to-variables behavior in the same dropdown.

## Out of scope

- The semantic/template-based simulation itself — explicitly called out by the ticket as a distinct, more technical feature, to be scoped and built separately.
- Any change to the existing "Sync with local styles" / "Sync with local variables" actions beyond however "Apply to selection" is placed alongside them in the same dropdown.
- Applying across multiple themes at once in a single action — only the active theme is assumed in scope (see Rules and Open questions).
- Reverse-engineering compound/gradient/image fills into a single "detected color" — traversal depth and fill-type handling aren't specified by the ticket (see Open questions).

## Implementation notes

No implementation exists yet. What has to be built, concretely:

- **Dropdown plumbing**: this doesn't introduce a new top-level mode (unlike [Structure](structure.md)) — it's a new entry in Edit mode's existing top-bar dropdown. `Actions.tsx` already renders `SYNC_LOCAL_STYLES`/`SYNC_LOCAL_VARIABLES`/`SYNC_LOCAL_TOKENS` as individually feature-gated entries in that menu. The Figma mockup shows the dropdown labeled "Apply to document…" with "Apply to selection" as a new first item, followed by "Sync with local styles"/"Sync with local variables" — whether this is the existing sync menu relabeled + extended, or a new separate menu, needs confirming against `Actions.tsx`'s actual current button label (not read in this session) — see Open questions.
- **New feature flag / message**: needs a new `Feature` entry (e.g. `APPLY_TO_SELECTION`) mirroring `SYNC_LOCAL_STYLES`'s gating pattern, and a new message type (e.g. `APPLY_TO_SELECTION`) dispatched from the UI to the host bridge.
- **New per-platform bridge work**: unlike the shared `ui-ui-color-palette/src/bridges/`, this needs a **new bridge in each host repo** (`figma-ui-color-palette`, `penpot-ui-color-palette`, `sketch-ui-color-palette`, `framer-ui-color-palette`) to read the current canvas selection via that platform's own API (`figma.currentPage.selection` and equivalents), walk its fills/strokes, and write back replacement colors. No existing bridge does selection/paint introspection today — `gets/getPalettesOnCurrentPage.ts` / `gets/jumpToPalette.ts` are palette-lookup bridges, not paint readers. See [Bridge catalog](../03-platform-bridges/bridge-actions.md).
- **Nearest-shade matching**: no engine change needed. `PaletteDataShadeItem` (`data.types.ts`) already exposes `hex` (and every other color-space representation) per shade; luminance for the detected canvas color and each candidate shade can reuse the same approach contrast scoring already uses elsewhere (`chroma(color).luminance()`, see `engine-ui-color-palette/src/modules/contrast/contrast.ts` / `color.ts`) — but per the ticket, the matching itself runs client-side in the UI/bridge layer, not through an engine/API call.
- **Reversibility**: no existing mechanism identified for "revert a selection to its pre-apply colors." Figma's own sync/generation bridges rely on `scheduleSaveVersion` (native version history) as their undo-safety net — whether this action can lean on the same native undo stack (Cmd/Ctrl+Z) or needs its own explicit "revert"/snapshot mechanism is unresolved (see Open questions).
- **Broader editor surface than Structure**: the ticket's editor list (Figma, FigJam, Buzz, Penpot, Sketch, Framer) is wider than [Structure](structure.md)'s (which excluded FigJam/Buzz) — this action is expected to work in FigJam (boards, not file canvases) and Buzz (browser-based editor), which likely need their own selection/paint APIs distinct from Figma's file-canvas one.
- **Analytics / Credits**: not specified by the ticket — to define alongside the build (a `trackActionEvent`-style event, and whether this consumes credits the way document generation/sync already do, see [Actions](actions.md)).

## Locales

Not yet checked against the Tolgee project (`UI Color Palette・Plugins`, id `2`) in this session — the Figma mockup's dropdown text ("Apply to selection," plus the button label itself, "Apply to document…") should be searched there before creating new keys, following the same process used for [Structure](structure.md)'s copy inventory.

| Text as seen in the mockup | Where | Tolgee key | Status |
| --- | --- | --- | --- |
| "Apply to selection" | New dropdown entry | — | Unread — not yet searched in Tolgee |
| "Sync with local styles" | Existing dropdown entry | `actions.syncLocalStyles` | Likely **reuse** (exists already, see [Structure](structure.md)'s inventory for the sibling `actions.syncLocalVariables`) |
| "Sync with local variables" | Existing dropdown entry | `actions.syncLocalVariables` | **Reuse** — confirmed to exist (see [Structure](structure.md)) |
| "Apply to document…" (button label, if new/renamed) | Top-bar dropdown trigger | — | Unread — not yet searched in Tolgee, and not yet confirmed whether this label is new or an existing one being reused |

## Open questions

- Is "Apply to document…" the existing sync menu (styles/variables/tokens) relabeled and extended with "Apply to selection," or a new, separate menu? The mockup only shows 2 of the 3 known sync items (styles, variables) alongside it — is "tokens" omitted for this specific mockup/platform, or dropped from this menu entirely?
- Reversibility mechanism: native undo only, or a dedicated "revert to original" action/toggle?
- Does the action match against the active theme only, or can the user pick which theme's shades to match against (a "Light ▾" dropdown appears nearby in the mockup, but it's [Preview](preview.md)'s existing theme switcher — unclear if this action reads it)?
- What counts as "multiple colors" and how deep does traversal go — nested components/instances, gradients, images (likely skipped, flat fills only?) — not specified by the ticket.
- Credits/plan gating: is this plan-gated/credit-consuming like sync-to-styles/variables, or free for all plans? Not specified by the ticket.
- Exact Tolgee key names — not yet searched in this session (see Locales).

## See also

- [Actions](actions.md) — the top-bar dropdown this new entry plugs into, and the existing `SYNC_LOCAL_STYLES`/`SYNC_LOCAL_VARIABLES`/`SYNC_LOCAL_TOKENS` gating pattern it should follow
- [Color System (feature)](color-system.md) — the separate, future semantic/template-based simulation this quick mode is explicitly kept independent from
- [Structure](structure.md) — the sibling in-progress spec this was drafted alongside; source of the confirmed `actions.syncLocalVariables`/`actions.syncLocalStyles` Tolgee keys reused here
- [`03-platform-bridges/bridge-actions.md`](../03-platform-bridges/bridge-actions.md) — the shared bridge catalog; this feature needs new, platform-specific bridges not yet in it
- [Palette](../01-domain-model/palette.md) — `PaletteDataShadeItem`'s `hex`/color-space fields this action reads to find the nearest shade

## History

| Date | Change |
| --- | --- |
| 2026-08-05 | Created as Draft — scopes "Apply to selection" (Notion ticket POR-645, "Simulation rapide de l'application de la palette sur un élément sélectionné") from the ticket's acceptance criteria and the Figma mockup (`UICP Screens`, node `4972:29275`). No existing UI/bridge code found for this (confirmed via grep: no `APPLY_TO_SELECTION`, no luminance-based palette matching anywhere in the codebase) — genuinely unbuilt, several open questions flagged pending clarification (exact dropdown composition, reversibility mechanism, theme scoping, traversal depth). |
