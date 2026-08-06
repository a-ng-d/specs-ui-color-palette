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
- **Theme-scoping — confirmed**: matches against the palette's **currently active theme's** shades only, not all themes at once.
- **Traversal scope — confirmed, no exceptions**: every object within the selection, at any nesting depth (nested components/instances included), whose fill or border/stroke is a solid or gradient color is processed — in short, every hex/RGB value found. Gradients are in scope: each color stop is matched and replaced individually, not skipped. Image fills are naturally out of scope, since they carry no solid/gradient color value to read.
- **Reversibility — confirmed**: relies entirely on the platform's **native undo** (Cmd/Ctrl+Z) — no dedicated in-app "revert to original" action or stored snapshot.
- **Plan gating — confirmed**: this is a **Pro** feature, consistent with the existing "Sync with local styles/variables/tokens" entries in the same dropdown. Unlike those three, it's **plan-gated only — no credit consumption**: confirmed free to run repeatedly once on a Pro plan.
- **Dropdown — confirmed**: "Apply to document…" is the existing "Synchronize…" dropdown (`actions.sync`), **renamed**, with "Apply to selection" added as a new first entry above the three existing sync actions (styles/variables/tokens) — not a new, separate menu.

## Acceptance criteria

- [ ] Given a selected element (page, illustration, or small component) containing multiple colors, when the user activates "Apply to selection," then every object in the selection, at any nesting depth, with a solid or gradient fill/border is replaced by a palette color from the active theme, chosen by closest luminance match — with no exceptions for depth or object type.
- [ ] Given a gradient fill or border, when "Apply to selection" runs, then each of its color stops is matched and replaced individually, not skipped.
- [ ] Given the quick simulation runs, then luminance computation and palette mapping happen entirely client-side, with no server/API call.
- [ ] Given the quick simulation's replacement, then no semantic role (background/text/border/state) is considered — luminance is the sole criterion.
- [ ] Given a quick-simulation result, when the user triggers undo (Cmd/Ctrl+Z), then the selection returns to its original colors — no dedicated in-app "revert" action exists or is needed.
- [ ] Given the semantic/template-based simulation (a separate, future feature), then this quick mode neither interferes with it nor shares its implementation or state.
- [ ] Given a user on a non-Pro plan, when they try "Apply to selection," then it's blocked with a trial/upgrade prompt, consistent with the gating already applied to "Sync with local styles/variables/tokens" in the same dropdown.
- [ ] Given the renamed "Apply to document…" dropdown, then it still contains the three existing sync actions (styles/variables/tokens) unchanged, with "Apply to selection" added as a new first entry — no regression to the existing sync behavior.

## Out of scope

- The semantic/template-based simulation itself — explicitly called out by the ticket as a distinct, more technical feature, to be scoped and built separately.
- Any change to the existing "Sync with local styles" / "Sync with local variables" / "Sync with local tokens" actions beyond the dropdown rename and the new first entry.
- Applying across multiple themes at once in a single action — only the active theme is in scope.
- A dedicated in-app "revert" action or snapshot mechanism — reversibility relies entirely on the platform's native undo.
- Image fills — out of scope by construction, since they carry no solid/gradient color value for this action to read.

## Implementation notes

No implementation exists yet. What has to be built, concretely:

- **Dropdown plumbing — confirmed direction**: this doesn't introduce a new top-level mode (unlike [Structure](structure.md)) — the existing "Synchronize…" dropdown (Tolgee key `actions.sync`) in `Actions.tsx`, which already renders `SYNC_LOCAL_STYLES`/`SYNC_LOCAL_VARIABLES`/`SYNC_LOCAL_TOKENS` as individually feature-gated entries, gets relabeled "Apply to document…" and gains "Apply to selection" as a new first entry above the three existing ones. No new menu component needed, just a new gated entry plus the trigger-label rename.
- **New feature flag / message**: needs a new `Feature` entry (e.g. `APPLY_TO_SELECTION`) mirroring `SYNC_LOCAL_STYLES`'s Pro-gating pattern (`isBlocked()` check, trial/upgrade prompt), and a new message type (e.g. `APPLY_TO_SELECTION`) dispatched from the UI to the host bridge.
- **New per-platform bridge work**: unlike the shared `ui-ui-color-palette/src/bridges/`, this needs a **new bridge in each host repo** (`figma-ui-color-palette`, `penpot-ui-color-palette`, `sketch-ui-color-palette`, `framer-ui-color-palette`) to read the current canvas selection via that platform's own API (`figma.currentPage.selection` and equivalents), **recursively walk every descendant node** (confirmed: no depth exception), inspect each node's fills/strokes for solid or gradient paints, and write back replacement colors per matched shade. No existing bridge does selection/paint introspection today — `gets/getPalettesOnCurrentPage.ts` / `gets/jumpToPalette.ts` are palette-lookup bridges, not paint readers. See [Bridge catalog](../03-platform-bridges/bridge-actions.md).
- **Nearest-shade matching**: no engine change needed. `PaletteDataShadeItem` (`data.types.ts`) already exposes `hex` (and every other color-space representation) per shade of the active theme; luminance for the detected canvas color and each candidate shade can reuse the same approach contrast scoring already uses elsewhere (`chroma(color).luminance()`, see `engine-ui-color-palette/src/modules/contrast/contrast.ts` / `color.ts`) — but per the ticket, the matching itself runs client-side in the UI/bridge layer, not through an engine/API call. For gradients, each stop's color is matched independently (confirmed) rather than the gradient being treated as a single unit or skipped.
- **Reversibility — confirmed, simpler than assumed**: relies entirely on the platform's native undo stack (Cmd/Ctrl+Z) — no explicit "revert"/snapshot logic to build, unlike Figma's sync/generation bridges which additionally call `scheduleSaveVersion` for their own safety net (not needed here, since this action doesn't touch the palette or generate a document).
- **Broader editor surface than Structure**: the ticket's editor list (Figma, FigJam, Buzz, Penpot, Sketch, Framer) is wider than [Structure](structure.md)'s (which excluded FigJam/Buzz) — this action is expected to work in FigJam (boards, not file canvases) and Buzz (browser-based editor), which likely need their own selection/paint APIs distinct from Figma's file-canvas one.
- **Credits — confirmed Pro-gating only, no credit consumption**: this is a Pro feature, per the requester — same plan-gating pattern as `SYNC_LOCAL_STYLES`/`SYNC_LOCAL_VARIABLES`/`SYNC_LOCAL_TOKENS` (`isBlocked()` check, trial/upgrade prompt). Unlike those three, it does **not** consume credits — confirmed no `plan.credits.fees.applyToSelection`-style fee is needed; free to use as many times as wanted once the plan is active.
- **Analytics**: not specified by the ticket — to define alongside the build (a `trackActionEvent`-style event, mirroring how sync actions are tracked, see [Actions](actions.md)).

## Locales

Confirmed via the Tolgee MCP — project `UI Color Palette・Plugins` (id `2`). No key exists yet for "Apply to selection" or "Apply to document" (searched directly, zero matches both times) — new keys needed. The dropdown's current trigger label **does** already exist: `actions.sync` = "Synchronize…" — the strongest candidate to rename/replace with "Apply to document…", rather than adding a parallel new key, since it's this exact button being relabeled (confirmed in Rules). All three existing sync entries (`actions.syncLocalStyles`, `actions.syncLocalVariables`, `actions.syncLocalTokens`) stay as-is and unaffected.

| Text as seen in the mockup | Where | Tolgee key | Status |
| --- | --- | --- | --- |
| "Apply to selection" | New dropdown entry | — | New — searched, no existing key |
| "Apply to document…" | Dropdown trigger (renamed) | `actions.sync` (currently "Synchronize…") | **Reuse the key, update its value** — this is the same button being relabeled, not a new one |
| "Sync with local styles" | Existing dropdown entry, unchanged | `actions.syncLocalStyles` | **Reuse**, no change |
| "Sync with local variables" | Existing dropdown entry, unchanged | `actions.syncLocalVariables` | **Reuse**, no change |
| "Sync with local tokens" | Existing dropdown entry, unchanged | `actions.syncLocalTokens` | **Reuse**, no change |
| Trial/upgrade prompt for a blocked "Apply to selection" attempt | Shared upsell modal | `proPlan.trial.title`/`.message`/`.cta`/`.option` | **Reuse** — confirmed generic, not per-feature: this is the same shared trial/upgrade modal every blocked action in the app already triggers (see [Actions](actions.md)'s "generic `GET_TRIAL`/`GET_PRO` upsell on any blocked action"), no new copy needed |

## Open questions

- ~~Is "Apply to document…" the existing sync menu relabeled and extended, or a new menu?~~ — resolved: existing `actions.sync` dropdown, renamed, with a new first entry. See Rules.
- ~~Reversibility mechanism?~~ — resolved: native undo only.
- ~~Active theme only, or theme-selectable?~~ — resolved: active theme only.
- ~~Traversal depth / what counts as "multiple colors"?~~ — resolved: every object in the selection at any depth, every solid/gradient fill or border, no exceptions.
- ~~Credits/plan gating?~~ — resolved: Pro-gated, same pattern as the sync actions.
- ~~Exact Tolgee key names?~~ — resolved: searched, see Locales. `actions.sync` reused for the renamed trigger; "Apply to selection" needs a new key.
- ~~Does this also consume credits in addition to being Pro-gated?~~ — resolved: no, plan-gating only, free to run repeatedly once on a Pro plan.
- ~~Exact Tolgee key for the Pro/upgrade-prompt copy?~~ — resolved: reuses the shared generic `proPlan.trial.*` upsell modal, no new copy. See Locales.

*(none remaining — all open questions from this spec have been resolved as of 2026-08-06)*

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
| 2026-08-06 | Resolved all six open questions: the dropdown is the existing "Synchronize…" menu (`actions.sync`) renamed to "Apply to document…" with a new first entry, not a new menu; reversibility is native undo only, no dedicated revert action; matches the active theme only; traversal is every object in the selection at any depth, every solid/gradient fill or border, no exceptions (gradient stops matched individually); confirmed Pro-gated. Searched Tolgee directly: no existing key for "Apply to selection" (new key needed) or "Apply to document" (reuses `actions.sync`, value updated); the three existing sync-entry keys are unaffected. Remaining open point: whether this also consumes credits in addition to being Pro-gated. |
| 2026-08-06 | Confirmed no credit consumption — plan-gating (Pro) only. Confirmed the trial/upgrade prompt reuses the app's shared generic `proPlan.trial.*` modal, no per-feature copy needed. No open questions remain in this spec. |
