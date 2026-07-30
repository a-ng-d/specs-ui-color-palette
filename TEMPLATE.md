# [Feature name]

- **Status**: Stub | Draft | Proposed | Accepted | Implemented | Superseded
- **Package(s) concerned**: e.g. `ui-ui-color-palette`, `engine-ui-color-palette`
- **Author**:
- **Date**:
- **Related spec**: (link to another spec if there's a dependency/succession)

## 1. Context

Why this spec exists. What user or technical need triggers this development. Link to the originating issue/discussion if one exists.

## 2. Current behavior (as-is)

What the code does today, only within the scope relevant to this feature. Don't re-audit the whole module — point to the relevant files (`src/ui/modules/...`, `src/bridges/...`, `src/stores/...`) rather than copying their content.

## 3. Proposal (to-be)

What changes. Target behavior, design decisions, edge cases handled.

## 4. Data model

Types/interfaces added or changed (reference `01-domain-model/` where possible rather than duplicating).

## 5. Impact

- **Bridges** (`src/bridges/`): new actions? modified actions?
- **Stores** (`src/stores/`): new state? migration of existing state?
- **Platforms** (Figma / Penpot / Sketch / Framer): is the feature available everywhere or only on certain platforms? Why?
- **API / MCP** (`04-contracts/`): new endpoint, new tool, or reuse of an existing one?
- **Credits / plan** (`config.fees`, `checkCredits`): does the action consume credits?
- **Analytics** (`types/events.ts`): new Mixpanel event to add?

## 6. Open questions

Undecided points. A `Proposed` spec can have open questions; an `Accepted` spec shouldn't have any blocking ones left.

## 7. History

| Date | Change |
| ---- | ------ |
|      | Created |
