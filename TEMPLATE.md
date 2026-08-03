# [Feature name]

- **Status**: Draft | Proposed | Accepted | Implemented | Superseded
- **Package(s) concerned**: e.g. `ui-ui-color-palette`, `engine-ui-color-palette`
- **Related spec**: (link to another spec if there's a dependency/succession)

## Problem

What's broken, missing, or painful today, from the user's or the product's perspective. Not a solution yet — just the gap this spec closes. Link to the originating issue/discussion if one exists.

## User flow

The steps a user actually takes, start to finish, once this ships. Numbered, concrete, in plain language — not a UI wireframe.

## Rules

The business rules and edge cases that govern the behavior: validation, limits, what happens when inputs conflict, platform-specific variations. This is where "what if X" gets answered before someone has to guess while implementing.

## Acceptance criteria

- [ ] Given <context>, when <action>, then <observable outcome>

One behavior per line, phrased so it can actually be checked by trying it. If a statement can't be verified this way, it belongs in Rules instead.

## Out of scope

What this spec deliberately does not cover, so nobody expands it mid-implementation or files a bug for something that was never promised.

## Implementation notes

Technical detail relevant to building this — reference `01-domain-model/` rather than duplicating its content:

- **Bridges** (`src/bridges/`): new actions? modified actions?
- **Stores** (`src/stores/`): new state? migration of existing state?
- **Platforms** (Figma / Penpot / Sketch / Framer): available everywhere or only on certain platforms? Why?
- **API / MCP** (`04-contracts/`): new endpoint, new tool, or reuse of an existing one?
- **Credits / plan** (`config.fees`, `checkCredits`): does the action consume credits?
- **Analytics** (`types/events.ts`): new Mixpanel event to add?

## See also

- Related specs, linked both ways.

## History

| Date | Change |
| ---- | ------ |
|      | Created |
