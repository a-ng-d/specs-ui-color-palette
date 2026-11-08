# Feature — Usage-based credits (metering)

- **Status**: In progress
- **Package(s) concerned**: `ui-ui-color-palette` (config, metering utility, gating, display, renewal); per-platform host repos (`figma-`, `penpot-`, `sketch-`, `framer-ui-color-palette`) for the canvas-side actions that must report what they actually did (sync, documentation, apply to selection)
- **Related spec**: [Structure AI](structure-ai.md) (first consumer: Mistral tokens), [Apply to selection](apply-to-selection.md), [Actions](actions.md) (sync and documentation actions), [User context](../01-domain-model/user-context.md) (credits fields)

## Problem

The credits system charges a **fixed fee per action**: 300 for any styles/variables/tokens sync, 200 for any generated document, 50 for an AI generation, and so on. The fee is the same whether the action touches 2 colors or 40. That has two consequences:

- small palettes and small selections overpay, while large ones underpay;
- actions whose real cost is variable, starting with LLM calls (billed by Mistral per token), can't be priced fairly at all.

The Structure AI work introduced a first answer: convert the **tokens actually consumed** into credits (see [Structure AI](structure-ai.md)). That mechanism isn't specific to AI. Any action can be metered by what it actually produces:
- colors synced to the local library;
- colors laid out in a document;
- layers recolored on a selection.

This spec generalizes it into one reusable metering model. Other workstreams can then adopt it, independently of Structure AI.

> **Product context.** Credits are coming back, and this metering model is how they come back. Until rollout, `VITE_CREDITS_ENABLED` stays `false`, and the public documentation still says the opposite ("Credits are gone, and so are per-action fees", `docs/general/introduction/free-and-pro.md`): it must be rewritten when credits ship. Every credit behavior described here, like the existing ones, stays behind `plan.isProEnabled && plan.isCreditsEnabled`. With the flag off, the metering still **measures** (for analytics and calibration) but neither blocks nor charges.

## User flow

1. Before a metered action, its trigger (button or menu item) shows an **estimate**: "≈ 24 credits", computed from what is already known (palette size, selection size, prompt size).
2. If the balance is below the estimate, the action is **blocked**, with the usual trial/Pro routing, as for fixed fees today.
3. The action runs. The platform (or Mistral) reports **what it actually produced**.
4. The **actual** cost is computed and charged. Feedback states it ("Synced 6 colors × 2 modes — 24 credits").
5. Once the balance reaches 0, it stays at 0 until the monthly renewal.

## Rules

### Allowance

- **500 credits per month**, renewed every month (today's code renews 500 every 7 days — to change, see Implementation notes).
- The allowance is meant to **encourage trying everything**: AI, simulation and exploration stay affordable, while the actions that produce a durable, one-shot output in the user's file — **sync with the local library** and **documentation** — cost more per color.

### Metering model

- A **meter** belongs to one action and defines:
  - its **unit** (what is counted) and how many **credits per unit** it costs — or, for AI, a tokens-to-credits conversion;
  - an optional per-run **floor** (`min`);
  - an optional per-run **cap** (`max`). **No meter uses a cap** (see Stop limit) — the field exists so one can be added later through configuration only.
- Cost of a run: `credits = ceil(Σ creditsPerUnitᵢ × unitsᵢ)`, raised to `min` when set, lowered to `max` when set.
- Units are counted from **what actually changed**, never from what was merely scanned. An up-to-date color or an unchanged layer counts 0.
- **Two phases**:
  - **estimate** before the run, from known inputs, used to block and display;
  - **settle** after the run, from measured units, which is what is charged.
- If the measured cost exceeds the estimate, the actual cost is still charged; the balance is clamped at 0, so a user is never pushed into debt.
- A failed run charges nothing. A partially successful run charges only what was measured (e.g. AI auto-bind charges the chunks that succeeded).
- Zero units → 0 credits, even when the meter has a floor.

### What a "color" is

For sync, documentation and updates, the unit is **one palette color in one mode**: a source color of the palette (with all its shades) × each mode (theme). A 6-color, 2-mode palette = 12 colors.

- **Sync**: a color counts when at least one of its shades was created, updated or deleted (deep sync) in that mode. Unchanged colors count 0. Deleting a color that was removed from the palette counts like any other change.
- **Documentation**: every color laid out in the document, per mode, whatever the format (palette, palette with properties, sheet).
- **Document updates**: the colors × modes actually regenerated. An update is a regeneration, so it costs the same per color as generating.

### Meters

| Action | Unit | Rate | Estimate from | Today (fixed) |
| --- | --- | --- | --- | --- |
| Sync styles | style (one shade in one theme) created, updated or deleted | **2 credits** | shades × colors × modes | 300 |
| Sync variables | variable × mode value created or updated; a deleted variable counts once per mode it had | **4 credits** | shades × colors × modes | 300 |
| Sync tokens (Penpot) | token created, updated or deleted | **4 credits** | shades × colors × modes | 300 |
| Document generation | palette color × mode laid out | **4 credits** (simple palette), **6** (detailed palette, ×1.5), **8** (color sheet, ×2) | colors × modes | 200 each |
| Push document updates / view switch | palette color × mode regenerated | rate of the regenerated document type (4 / 6 / 8) | colors of the selected document (1 mode) | 100 |
| Apply to selection (simulation) | layer whose color actually changed (fill, stroke or shadow) | **2 credits** | layers in the selection | — |
| AI auto-binding (Structure) | tokens: input × 1 + output × 4 | **1 credit per 400 weighted tokens**, floor 10 | context size + targets × 60 | — |
| AI Structure assistant (per message) | same | same, floor 5 | context + history + prompt + instructions | — |
| AI color generation | same | **1 credit per 40 weighted tokens** (10× the Structure AI rate), no floor | prompt size + instructions + ≈350 output tokens | — (metered) |

Worked examples on a 6-color, 2-mode palette:
- first full sync of variables: 12 colors × 2 = **24 credits**; re-syncing after editing 2 colors: 2 × 2 × 2 = **8 credits**; re-syncing with nothing changed: **0**;
- generating any document: **24 credits**;
- simulating on a selection where 40 layers change color: **40 credits**.

AI meters are already implemented (`plan.aiCredits`, 400 tokens per credit, output ×4, floors 10 and 5). The code still sets caps (250 and 100), which must be removed (see Stop limit).

Meters for canvas actions need the **platform bridge to report counts** back to the UI (see Implementation notes). Until a platform reports them, that action keeps its fixed fee.

### Stop limit

- **No caps.** No action has a per-run ceiling: capping is a risk (a large run would be underpriced, and caps are easy to game by batching). What protects the allowance is the per-color pricing, the estimate shown before running, and the balance block.
- The `max` field stays in the model so that a cap can be **added or removed per action through configuration only**, should the data show a need.
- **Balance block** applies to every meter: estimate > balance → the action is blocked before starting.
- **Balance floor**: the balance never goes below 0. When the actual cost exceeds the remaining balance, the balance drops to 0 and stays there until renewal. No debt, no carry-over.
- No action stops half-way because of credits: once started, a run completes and is charged its actual cost.

### Rollout

- **The principle is settled; the numbers are provisional.** Rates, floors and the allowance above are starting values, adjusted from the product owner's own tests before and after rollout.
- **Phase 1 — no caps, measure.** Nothing is capped, so the real cost of each action shows up unaltered in the measurements. Caps are only considered afterwards, if the data calls for them.
- **Order of adoption**:
  1. **Document generation** — first canvas meter, delivered by the upcoming (higher-priority) workstream that already builds on this spec;
  2. document updates and sync (styles / variables / tokens);
  3. apply to selection, once that feature exists ([Apply to selection](apply-to-selection.md) is still a draft).

  AI actions are already metered (Structure AI); AI color generation moves from its fixed fee whenever convenient.

### Calibration

- Every metered run tracks its measured units and charged credits (analytics), **including while credits are disabled**, so rates can be checked against real distributions.
- Rates are checked against the monthly allowance: a month of exploration (a few palettes, some AI, simulations, a couple of syncs and documents) should fit in 500 credits; heavy production use should lead to Pro.

### Display

- Every metered trigger shows its estimate. Every result shows the actual cost. Both only when credits are enabled.
- The plan / credits panel (`PlanControls`) describes each meter by its rule ("2 credits per color and mode", "1 credit per 400 tokens, output ×4"), not by a fixed number.

## Acceptance criteria

- [ ] Given a meter, when a run produces units U, then the charge equals `ceil(Σ rateᵢ × Uᵢ)` (raised to the floor when one is set), and 0 when U = 0.
- [ ] Given a 6-color, 2-mode palette, when variables are synced for the first time, then 24 credits are charged; when any document format is generated, then 24 credits are charged.
- [ ] Given a sync where 2 colors changed in both modes, then 8 credits are charged; given nothing changed, then 0.
- [ ] Given a deep sync that deletes a color removed from the palette, then that color counts like any other change.
- [ ] Given a simulation on a selection where 40 layers change color, then 40 credits are charged.
- [ ] Given any action, then no cap limits its cost; given a cap added in configuration, then it applies without any code change.
- [ ] Given a balance below the estimate, when the user triggers the action, then it is blocked and routed to trial/Pro.
- [ ] Given an actual cost above the remaining balance, then the balance drops to 0 and stays there until the monthly renewal.
- [ ] Given a started run, then it completes and is charged, or fails and charges nothing — credits never stop it half-way.
- [ ] Given credits disabled, then no action is blocked or charged, and measured units are still tracked.
- [ ] Given a platform that doesn't report counts yet for an action, then that action keeps its fixed fee.
- [ ] Given the renewal date is reached, then the balance is reset to 500 and the next renewal is set one month later.

## Out of scope

- Server-side metering and tamper resistance: the balance lives in client storage (`credits_count` in `localStorage` / plugin storage) and can be edited by the user, as it can today. Moving it server-side is a separate project.
- Billing, top-ups or purchasable credit packs.
- Rewriting the public docs and the plans page (to do at rollout, see Product context).

## Implementation notes

- **Allowance and renewal** — `plan.creditsLimit` stays 500. `plan.creditsRenewalPeriodDays` / `creditsRenewalPeriodHours` (`ui-ui-color-palette/src/global.config.ts`, today 7 days / 168 h) move to a monthly period. A calendar month is more natural than a fixed 30 days: `checkCredits` (`src/bridges/checks/checkCredits.ts`) computes the next renewal with `addHours`, and would need an `addMonths` for that. Bump `versions.creditsVersion` so every client is reset onto the new allowance once.
- **Config** — generalize `plan.aiCredits` (typed in `src/types/config.ts`) into `plan.meters`, keyed by action:

  ```ts
  type Meter = {
    units: Record<string, number>   // unit kind -> credits per unit, e.g. { color: 2 } or { layer: 1 }
    tokens?: {                      // AI only: tokens-to-credits conversion
      tokensPerCredit: number       // 400
      outputTokenWeight: number     // 4
    }
    min?: number                    // floor
    max?: number                    // cap — unused by default, add through configuration only
  }
  ```

  Migration: `aiCredits.bindings` → `meters.aiBindings`; `aiCredits.structureChat` → `meters.aiStructureChat`; remove their `max`. A canvas action moves its entry from `fees` to `meters` only once its platform reports counts.
- **Utility** — generalize `src/utils/computeAiCost.ts` (`computeAiCost`, `estimateAiCost`, `sumUsage`) into `computeMeteredCost(meter, usage)` / `estimateMeteredCost(meter, estimatedUsage)`, with `usage: Record<unitKind, number>`. AI calls map `promptTokens` / `completionTokens` through `tokens`.
- **Gating** — keep `FeatureStatus.isReached((creditsCount − estimate) × −1 − 1)` with `limit: 0` on the feature, as for fixed fees today. Credits are decremented through `$creditsCount` (`src/stores/credits.ts`), clamped at 0 and persisted by `App.tsx`.
- **Message contract** — one generic Canvas → UI message settles every metered canvas action: `{ type: 'REPORT_USAGE', data: { action, usage } }`, where `action` is the triggering UI → Canvas type (`SYNC_LOCAL_STYLES`, `SYNC_LOCAL_VARIABLES`, `SYNC_LOCAL_TOKENS`, `CREATE_DOCUMENT`, `UPDATE_DOCUMENT`, `SIMULATE_PALETTE`). It is posted once per successful run and never on failure (nor when a simulation has an empty selection). `usage.color` is the number of **distinct (source color × mode) pairs** created, updated, deleted or laid out — not `colors × modes`, which would overcharge when different colors change in different modes. `usage.layer` is the number of layers whose color actually changed (fill, stroke or shadow; one layer counts once). The simulation estimate comes from a second message, `{ type: 'SELECTION_MEASURED', data: { layers } }`, posted on every selection change and at start. Each platform lists the actions it reports in `meteredActions` (host and UI configs); an action outside that list keeps its fixed fee, charged on click. Wired on Figma, Penpot, Sketch and Framer.
- **Analytics** — the existing action events (`trackActionEvent` and siblings) gain `units` and `creditsCharged` properties.
- **Today in code** — AI meters are live behind the credits flag (Structure AI), still with caps to remove. Canvas actions still use fixed `fees.*`: `src/ui/modes/EditPalette.tsx` (sync and documentation), `src/ui/modules/Actions.tsx` (document menu labels).

## Locales

| Text | Where | Tolgee key | Status |
| --- | --- | --- | --- |
| "≈ {cost} credits" on a trigger | Any metered button/menu item | `credits.estimate` | New (candidate — AI features currently use their own `…WithCost` keys) |
| "{cost} credits used" after a run | Result feedback | `credits.charged` | New (candidate) |
| Meter rule in the plan panel ("{rate} credits per color and mode", "1 credit per {tokens} tokens") | `PlanControls` | `plan.credits.fees.*` | Reuse, reworded per meter |
| Monthly renewal date | `PlanControls` | existing renewal keys | Reuse, check wording ("every week" → "every month") |

Keys pending review (keep / rename / drop) before creation in Tolgee.

## Open questions

- **AI vs. durable actions balance** (to settle from the tests): at 400 tokens per credit, a full auto-bind of the 128-token example costs about 95 credits and an assistant message 20–35, i.e. more than a full sync or document of a 6-color palette (24). That goes against "sync and documentation cost more". Should the AI rate be relaxed (e.g. 1,500 tokens per credit → auto-bind ≈ 25, message ≈ 5–9), or is the gap intended?
- **Public documentation**: when and how `free-and-pro` is rewritten to announce the return of credits.

## See also

- [Structure AI](structure-ai.md) — the first metered actions (Mistral tokens)
- [Apply to selection](apply-to-selection.md) — metered per recolored layer
- [Actions](actions.md) — sync and documentation entry points
- [User context](../01-domain-model/user-context.md) — `creditsCount`, `creditsRenewalDate`, `Config.plan`

## History

| Date | Change |
| --- | --- |
| 2026-11-05 | Created as Proposed: generalizes the token-based credits introduced by Structure AI into a reusable metering model, with two-phase estimate/settle, stop limits, candidate meters for apply to selection, sync and documentation, and a migration path from fixed fees. |
| 2026-11-05 | Decisions folded in: credits are coming back (public docs to be rewritten at rollout); balance floors at 0, no debt; no partial runs; deep-sync deletions count; documents metered as colors × modes whatever the format. |
| 2026-11-05 | Pricing decided: 500 credits renewed **monthly**; sync, documentation and updates at **2 credits per palette color and mode**; apply to selection at **1 credit per recolored layer**; AI through the tokens-to-credits conversion; **no caps** on any action (field kept for configuration only). Open question raised on the AI rate vs. durable actions. |
| 2026-11-05 | Rollout added: principle settled, numbers provisional and tuned from the product owner's tests; phase 1 with no caps at all, to measure real costs; document generation is the first canvas meter, delivered by the upcoming higher-priority workstream, then updates and sync, then apply to selection. |
| 2026-11-05 | Implementation started on all four hosts (Figma, Penpot, Sketch, Framer) for sync, documents, document updates and simulation: message contract fixed as `REPORT_USAGE` (single `color` count of distinct color × mode pairs) and `SELECTION_MEASURED`; per-platform `meteredActions` fallback; charge moved from click to settle for metered actions; analytics event fired at settle. AI meters still on `feat/add-ai-structure`, migrated to `plan.meters` when it merges. |
| 2026-11-05 | Pricing revised after the product owner's first tests (too cheap): sync is now metered per written item — **1 credit per style**, **per variable × mode**, **per token** (≈120 credits for a first sync of 6 colors × 10 shades × 2 modes); documents and updates **4 credits per color × mode**; simulation **2 credits per changed layer**. Credit-gated Pro features (documents, sync, simulation) are now usable by free users while credits remain, blocked only when out of credits. Worked examples above still reflect the first pricing and must be refreshed. |
| 2026-11-05 | Fixed fees removed entirely (config `fees` and the per-platform fixed-fee fallback `meteredActions`): Coolors, Realtime Colors and Color Hunt imports, color harmony, image color extraction and palette creation no longer cost anything; only usage-based actions are charged. AI color generation is metered on Mistral tokens. Credits apply to the free tier only (no deduction for Pro). Amounts are displayed as **cdts** ("≈ 48 cdts", "1 cdt per style"). |
| 2026-11-05 | AI color generation raised ×10: 1 credit per **40** weighted tokens (input ×1, output ×4), ≈ 40–50 cdts per generation instead of ≈ 4–5. Pro offers list "Unlimited credits" in every locale; estimates are hidden for Pro users. |
| 2026-11-05 | Document price now depends on its type: simple palette 4, detailed palette 6 (×1.5), color sheet 8 (×2) cdts per color × mode; updates and view switches use the rate of the regenerated type. The canvas still reports `{ color }` only — the UI resolves the meter from the view recorded at click. Plan panel rule lines moved to `plan.credits.meters.<MeterKey>` (old `plan.credits.fees.*` removed from every locale) to stop stale remote/translated strings using `{fee}` from rendering "invalid". |
| 2026-11-05 | Sync rates raised: **2 cdts per style**, **4 per variable × mode**, **4 per token** (≈240 cdts for a first styles sync, ≈480 for variables, on 6 colors × 10 shades × 2 modes). Tolgee: new `credits.*` and `plan.credits.meters.*` keys created, every `plan.credits.fees.*` key tagged `Deprecated`. The UI web harness now reports mock usage (first-run counts) so credits can be exercised in the browser. |
