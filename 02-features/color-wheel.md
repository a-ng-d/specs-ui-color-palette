# Feature — Color wheel (harmony-based palette seeding)

- **Status**: Implemented
- **Package(s) concerned**: `ui-ui-color-palette`
- **UI module**: `src/ui/services/ColorWheel.tsx`
- **Related spec**: [Creation](creation.md) (overview, shared cost model), [Colors](colors.md)

## Problem

One of four alternative ways to seed a new palette: pick a base color and a harmony rule, get a matching set of source colors. It works as expected; no confirmed bugs identified.

## User flow

1. The user picks a base color and one of five harmony rules: Analogous, Complementary, Compound, Triadic, Tetradic — each individually plan-gated.
2. The wheel recomputes live as either the base color or the rule changes, showing a preview row of the resulting swatches (hex code + closest color name per swatch).
3. The user commits with "Use this palette" — the candidate colors become the new palette's source colors, and the app switches back to the Manage service with the new palette already created and open.

## Rules

- The harmony math itself (`ColorHarmony` class — analogous/complementary/compound/triadic/tetradic generators) lives outside this component; `ColorWheel.tsx` only owns the base color, the selected rule, and re-triggers computation on change.
- The base color defaults to a fixed light cyan (the same default color used to seed a new source color in Colors — `rgb(0.533, 0.921, 0.976)`), not a random one.
- Generated source colors are tagged `source: 'HARMONY'`, with hue/chroma shift both reset to 0 (not locked) — distinct from Colors' own "add color" default, which seeds chroma shift at 100.
- Follows the shared two-fee cost model documented in [Creation](creation.md): `harmonyCreate` is debited when generating/previewing, `paletteCreate` when committing.

## Acceptance criteria

- [ ] Given a harmony rule change, when the wheel recomputes, then the preview updates without a page reload or explicit "generate" click — it's live.
- [ ] Given the local-palette quota is reached, when the user tries to commit, then the action is blocked with a trial/upgrade prompt.
- [ ] Given a harmony rule the user's plan doesn't include, when they try to select it, then it's blocked with a trial/upgrade prompt, independently of the other four rules.

## Out of scope

- Adding new harmony rules or changing the underlying color-harmony math.
- Randomizing the default base color.

## Implementation notes

Reuses the engine's `SourceColorConfiguration` shape; harmony computation is delegated to a `ColorHarmony` helper instantiated once in the constructor and mutated in place (`setBaseColor`) rather than recreated per change.

- **Stores**: reads `$palette` only to attach the current in-progress exchange data to the `CREATE_PALETTE` message; writes nothing itself (see Creation for how the actual palette gets created).
- **Bridges**: sends `CREATE_PALETTE` with `{ sourceColors, exchange }` — same shape as the other three creation methods.
- **Analytics**: `CREATE_COLOR_HARMONY` (commit) via `trackImportEvent`; the underlying `CREATE_PALETTE` action itself is tracked separately via `trackActionEvent`.
- **Credits**: `harmonyCreate` (generation) + `paletteCreate` (commit) — see Creation's shared cost model.
- **Platforms**: none identified.

## Open questions

*(none)*

## See also

- [Creation](creation.md) — the shared overview, cost model, and the three sibling methods
- [Colors](colors.md) — the analogous per-color hue/chroma shift and default-seed-color behavior in the main editor

## History

| Date | Change |
| --- | --- |
| 2026-08-04 | Created — as-is consolidation from reading `ColorWheel.tsx`, following up on a request to cover the creation services |
