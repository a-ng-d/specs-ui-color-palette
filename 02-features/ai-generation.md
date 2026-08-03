# Feature — AI generation (prompt-based palette seeding)

- **Status**: Implemented
- **Package(s) concerned**: `ui-ui-color-palette`
- **UI module**: `src/ui/services/GenAI.tsx`
- **Related spec**: [Creation](creation.md) (overview, shared cost model), [Colors](colors.md)

## Problem

One of four alternative ways to seed a new palette: describe what you want in a free-text prompt, get a 5-role palette back from Mistral AI. It works as expected; the only confirmed gap is that a missing/unavailable AI client fails with a generic error rather than distinguishing "not configured" from "request failed."

## User flow

1. The user types a free-text prompt, or picks one of 12 curated presets (6 "vibes": cyberpunk, minimalist, pastel, corporate, nature, vintage; 6 "use cases": landing, blog, resume, portfolio, documentation, ecommerce) — hovering a preset previews its full prompt text before selecting it.
2. The user requests generation; while processing, the UI shows a loading state.
3. The AI returns a fixed 5-role palette — primary, text, success, warning, alert — each becoming one source color, named `"{role} · {AI-given name}"`.
4. The user commits with "Use this palette" — same handoff as the other three methods (see Creation): the palette is created and opened immediately.
5. If generation fails (including when the AI client isn't available at all), an inline error message is shown instead of a result.

## Rules

- Generation calls the Mistral client **directly from the UI** (`getMistral()`), not through a bridge/host round-trip — this is a client-side API call, unlike every other palette-mutating action in the app.
- The 5 roles are fixed (primary/text/success/warning/alert) — the AI doesn't return an arbitrary-length palette, always exactly these 5.
- Generated source colors are tagged `source: 'AI'`, with hue/chroma shift both reset to 0 (not locked) — same as Color wheel's output, different from Colors' own "add color" default (chroma shift 100).
- If `getMistral()` returns no client (not configured), generation fails immediately with the same generic error message as an actual failed request (`error.unavailableAi`) — no distinct "AI isn't available in this environment" message.
- Follows the shared two-fee cost model documented in [Creation](creation.md): `aiColorsGenerate` is debited on a successful generation, `paletteCreate` on commit.

## Acceptance criteria

- [ ] Given a preset prompt is hovered, when the user moves away without selecting it, then the prompt input reverts to showing nothing selected (preview only, not a commit).
- [ ] Given a generation request succeeds, when the result renders, then exactly 5 source colors are produced (primary/text/success/warning/alert), each named with its role and the AI-given color name.
- [ ] Given the AI client isn't configured, when the user requests generation, then this is distinguishable from a failed network request. *(currently both show the same generic error — confirmed gap)*
- [ ] Given the local-palette quota is reached, when the user tries to commit, then the action is blocked with a trial/upgrade prompt.

## Out of scope

- Changing the fixed 5-role output shape or supporting a variable number of generated colors.
- Distinguishing the "not configured" vs. "request failed" error message — flagged, not fixed here.
- Any change to the underlying Mistral prompt/model used for generation.

## Implementation notes

Reuses the engine's `SourceColorConfiguration` shape; the Mistral response type (`MistralColorPalette`) is converted to source colors via a dedicated mapping function preserving each role's AI-given name.

- **Stores**: reads `$palette` only to attach the current in-progress exchange data to the `CREATE_PALETTE` message; writes nothing itself.
- **Bridges**: sends `CREATE_PALETTE` with `{ sourceColors, exchange }` — same shape as the other three creation methods. The generation call itself bypasses the bridge entirely (direct client-side API call).
- **Analytics**: the commit action is tracked via `trackImportEvent`/`trackActionEvent`, same pattern as the other three methods; no distinct "generation requested" or "generation failed" event was identified.
- **Credits**: `aiColorsGenerate` (successful generation only — not charged on failure) + `paletteCreate` (commit) — see Creation's shared cost model.
- **Platforms**: none identified.

## Open questions

*(none)*

## See also

- [Creation](creation.md) — the shared overview, cost model, and the three sibling methods
- [Colors](colors.md) — the analogous default-seed-color and hue/chroma-shift behavior in the main editor

## History

| Date | Change |
| --- | --- |
| 2026-08-04 | Created — as-is consolidation from reading `GenAI.tsx`, following up on a request to cover the creation services |
