# Feature — Structure AI (auto-binding + Structure assistant)

- **Status**: Accepted
- **Package(s) concerned**: `ui-ui-color-palette` (UI, web bridge, Mistral client); platform repos (`figma-`, `penpot-ui-color-palette`) for persistence parity — follow-up
- **Related spec**: [Structure](structure.md), [AI generation](ai-generation.md), [Color System (domain model)](../01-domain-model/color-system.md)

## Problem

Binding a taxonomy by hand is tedious — the shipped example alone (Type × Prominence × State) yields 128 tokens, each needing a `<colorId>:<stop>` ref picked from a dropdown. Designing the taxonomy itself also requires design-system know-how many users don't have. [Structure](structure.md) v1 deliberately shipped hand-editing only; this spec adds two Mistral-backed aids on top, without taking control away from the user.

## User flow

### Auto-binding (Binding tab)

1. In the Binding tab, the user clicks the AI icon button next to the "Search tokens…" field.
2. The button goes into loading while a dedicated Mistral agent (`UICP Binding`) receives the palette (colors × stops × themes, with hex values), the taxonomy schema and the list of target tokens.
3. The returned refs are validated, merged into the bindings, written to `$system` and persisted (`UPDATE_SYSTEM_BINDINGS`). A message reports how many tokens were bound and how many suggestions were skipped.

### Structure assistant (drawer)

1. In Structure mode, the user opens the assistant from an icon button in the top bar. A resizable drawer opens on the right (bottom on mobile), like Edit mode's context drawer.
2. Past conversation for this palette is restored.
3. The user writes a message (or picks a suggested prompt). The `UICP Structure Assistant` agent receives a fresh snapshot of the palette, schema and bindings plus the recent conversation.
4. The reply is shown. If it carries a patch (schema and/or bindings changes), a summary is shown with **Apply** and **Discard**. Nothing changes until the user clicks Apply.
5. Applying updates Taxonomy/Binding/Visualize immediately and persists the system; the message is marked "Applied".
6. The user can clear the conversation.

## Rules

- **Auto-bind never overwrites a manual choice**: targets are tokens that are not excluded and have no ref (or `EMPTY_REF_VALUE`). If rows are multi-selected, targets are restricted to the selection. No target → no request.
- More than 150 targets are sent in sequential chunks with the same palette context.
- **Client-side validation is authoritative**: a suggestion is dropped if its path is not a target (auto-bind) / not a resolvable path (assistant), its `colorId` is unknown, or its stop isn't in `preset.stops`. Dropped entries are counted and reported, never silently applied.
- Merge keeps existing `description` and `isExcluded`; an AI description fills the field only when it's empty.
- **Assistant patches are proposals**: never auto-applied. A patch's `schema` is a full replacement; new groups/members use `new:<slug>` placeholder ids, replaced by real uids consistently across schema and binding paths. Bindings referencing removed members are pruned.
- **Stale patch**: if the system changed since the proposal, Apply still works — the patch is re-validated against the current state and skipped entries are reported.
- Conversation history is per palette, capped to the last 50 messages in storage; only the last 20 are sent to the agent. Deleting the palette deletes its history (and its system).
- Without a Mistral client or without the relevant agent id configured, both entry points are disabled with the existing `error.unavailableAi` helper.

## Acceptance criteria

- [ ] Given a taxonomy with unbound tokens, when the user clicks the auto-bind button, then every non-excluded unbound token receives a valid ref and the result survives a reload.
- [ ] Given manually bound or excluded tokens, when auto-bind runs, then they are left untouched.
- [ ] Given a multi-selection, when auto-bind runs, then only unbound tokens inside the selection change.
- [ ] Given no unbound target, when the button is clicked, then no request is sent and a neutral message is shown.
- [ ] Given the assistant returns a patch, then nothing changes until Apply is clicked; after Apply, Taxonomy/Binding/Visualize reflect it and the message shows "Applied".
- [ ] Given a patch referencing an unknown color or stop, when applied, then those entries are skipped and reported.
- [ ] Given a conversation, when the palette is reopened, then the history (including Applied/Discarded statuses) is restored.

## Out of scope

- Platform-side persistence on Figma/Penpot (`UPDATE_SYSTEM_*` and `*_STRUCTURE_CHAT` handlers in the platform repos) — tracked as a follow-up; in Figma the UI iframe has no reliable `localStorage` (use `figma.clientStorage` / plugin data on the Canvas side); Penpot storage is synchronous and string-only.
- Streaming responses.
- Server-side proxying of the Mistral API key (today it ships client-side, as for AI color generation) — recommended follow-up via `api-ui-color-palette`.
- Cloud sync of the conversation.

## Implementation notes

- **Message contract**:

  | type | Direction | Payload |
  | --- | --- | --- |
  | `UPDATE_SYSTEM_BINDINGS` | UI → Bridge | `{ id, data: TaxonomyBinding[] }` (existing) |
  | `UPDATE_SYSTEM_SCHEMA` | UI → Bridge | `{ id, data: TaxonomySchema }` (existing) |
  | `UPDATE_STRUCTURE_CHAT` | UI → Bridge | `{ id, data: StructureChatHistory }` |
  | `GET_STRUCTURE_CHAT` | UI → Bridge | `{ id }` |
  | `GET_STRUCTURE_CHAT` | Bridge → UI | `{ data: { id, history } }` |

- **Types**: `src/types/structureChat.ts` (`StructurePatch`, `StructureChatMessage`, `StructureChatHistory`), `src/external/mistral/types.ts` (`StructureAgentContext`, `MistralBindingSuggestion`, `MistralStructureReply`).
- **Bridges** (web): `palette_${id}_structure_chat` in `localStorage`; purged by `deletePalette` together with `palette_${id}` and `palette_${id}_system`.
- **Stores**: `$system` (existing), `$structureChat` (new atom).
- **Mistral client**: `callAgent(agentId, messages)` generalized; agent ids configurable via `VITE_MISTRAL_AI_COLORS_AGENT_ID`, `VITE_MISTRAL_AI_BINDINGS_AGENT_ID`, `VITE_MISTRAL_AI_STRUCTURE_AGENT_ID`; methods `suggestBindings(ctx)` and `chatStructure(ctx, history, message)`.
- **Feature flags**: `BINDING_AI`, `STRUCTURE_ASSISTANT`.
- **Analytics**: `trackActionEvent` with `feature: 'AI_BINDINGS' | 'AI_STRUCTURE_CHAT'`.

## Agent instructions (Mistral La Plateforme)

Two agents, `mistral-medium-latest` or `mistral-large-latest`, response format **JSON object**, temperature 0.2 (binding) / 0.4 (assistant). Versioned here so prompt changes stay reviewable.

### Agent A — `UICP Binding` → `VITE_MISTRAL_AI_BINDINGS_AGENT_ID`

```
You are a senior design-system engineer. You map semantic design tokens to primitive color shades.

INPUT: a single JSON object with:
- palette.colors[]: { id, name, description?, shades: { "<stop>": { "<themeName>": "#hex" } } }
- palette.stops[]: the available stops, ordered from lightest to darkest in the light theme
- palette.themes[]: the color modes (e.g. Light, Dark). One ref resolves per theme automatically.
- schema.groups[]: taxonomy dimensions (e.g. Type, Prominence, State) with their members
- tokens[]: the tokens to bind; each has path (member ids) and pathNames (human-readable, in group order)

TASK: for EVERY token in tokens[], choose exactly one primitive ref "<colorId>:<stop>".

RULES:
1. Infer intent from pathNames. Typical semantics:
   - Type: Background/Surface → light stops in the light theme; Text/Icon → high-contrast stops; Border → mid stops.
   - Prominence/Role: Primary/Brand/Accent → the brand-like color; Danger/Error → red-ish; Success → green-ish;
     Warning → amber/yellow; Info → blue-ish; Secondary/Tertiary/Neutral → the most desaturated or gray color.
     Match on color NAME first, then on hue from the hex values.
   - State: Default = base stop; Hover = one stop darker (light theme); Pressed = two stops darker; Disabled = a low-contrast,
     desaturated stop; Focus = same as Default or one stop stronger.
2. Keep siblings consistent: tokens that differ only by State should use the SAME colorId and adjacent stops.
3. Text/Icon tokens should aim for WCAG AA (≥ 4.5:1) against the Background token of the same Prominence in every theme.
4. Use ONLY colorIds from palette.colors and stops from palette.stops. Never invent ids. Never change path.
5. If no sensible color exists for a role, use the most neutral color rather than skipping the token.

OUTPUT: JSON only, no prose, no markdown:
{ "bindings": [ { "path": ["<memberId>", ...], "ref": "<colorId>:<stop>", "description": "<≤ 80 chars: what this token is for>" } ] }
```

### Agent B — `UICP Structure Assistant` → `VITE_MISTRAL_AI_STRUCTURE_AGENT_ID`

```
You are the Structure assistant of UI Color Palette. You help a designer build a semantic color system
(taxonomy + bindings) on top of their primitive palette, in a conversation.

Each turn begins with a message starting with "CONTEXT:" followed by JSON describing the CURRENT state:
palette (colors, stops, themes with hex values), schema (groups/members with ids), tokens (path, pathNames, ref, isExcluded).
This context is authoritative and replaces any earlier one. Then comes the conversation, then the user's new message.

CONCEPTS:
- A taxonomy is an ordered list of groups (dimensions); tokens are the cartesian product of their members.
- A binding maps a token path (array of member ids, in group order) to a ref "<colorId>:<stop>". Refs resolve per theme automatically.
- isExcluded removes a token everywhere. Unbound tokens are allowed.

BEHAVIOUR:
- Answer in the user's language, concisely (≤ 120 words). Explain the reasoning behind naming or color choices.
- Propose changes ONLY when the user asks for them, or when a fix is clearly implied. Otherwise patch = null.
- Ask a short clarifying question instead of guessing when the intent is ambiguous (patch = null).
- Watch out for combinatorial explosion: warn the user when a schema change produces more than 300 tokens.
- Binding heuristics: name/hue matching for roles, adjacent stops for states, AA contrast for Text/Icon over Background.

PATCH RULES:
- "schema", if present, is the FULL new schema. Keep the existing ids for unchanged groups/members.
  New groups/members use ids "new:<kebab-slug>", unique within the patch; reuse the same placeholder in binding paths.
- "bindings.upsert" entries are partial: include only the fields you change (ref, description, isExcluded).
- "bindings.remove" lists paths whose binding should be deleted (the token becomes unbound).
- Use only colorIds and stops present in the context. Never invent primitive colors.

OUTPUT: JSON only, no markdown:
{ "reply": "<message to the user>",
  "patch": null | { "schema"?: { "groups": [ { "id", "name", "members": [ { "id", "name" } ] } ] },
                    "bindings"?: { "upsert"?: [ { "path": [...], "ref"?: "...", "description"?: "...", "isExcluded"?: bool } ],
                                   "remove"?: [ [ ...path ] ] } } }
```

## Locales

| Text | Where | Tolgee key | Status |
| --- | --- | --- | --- |
| "Suggest bindings" | Binding tab, AI button helper | `structure.binding.ai.suggest` | New |
| Loading / success ("{count} tokens bound, {skipped} skipped") / no target | Binding tab | `structure.binding.ai.loading`, `.ai.success`, `.ai.noTarget` | New |
| "Structure assistant", placeholder, send, clear | Drawer | `structure.assistant.title`, `.placeholder`, `.send`, `.clear` | New |
| Apply / Discard / Applied / Discarded / patch summary / stale notice | Drawer, patch card | `structure.assistant.apply`, `.discard`, `.applied`, `.discarded`, `.patchSummary`, `.stale` | New |
| Suggested prompts | Drawer | `structure.assistant.presets.*` | New |
| Top-bar toggle helper | Actions | `contexts.assistant` | New |
| AI unavailable | Both | `error.unavailableAi` | Reuse |

Keys pending review (keep / rename / drop) before creation in Tolgee.

## See also

- [Structure](structure.md) — the mode this builds on
- [AI generation](ai-generation.md) — the existing Mistral flow this mirrors (client)

## History

| Date | Change |
| --- | --- |
| 2026-10-21 | Created — auto-binding (fill unbound only, selection-aware) and conversational Structure assistant (proposal + Apply), with message contract, persistence and Mistral agent instructions. |
| 2026-10-29 | Plan, gating and credits content removed; plans live in the docs (free-and-pro) |
