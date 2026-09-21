# Platform — Chrome Extension

- **Status**: Draft — concept only, no implementation yet
- **Host repo**: not yet created — proposed `chrome-ui-color-palette`, following the `<surface>-ui-color-palette` naming convention in [`00-overview/architecture.md`](../00-overview/architecture.md)
- **Related spec**: [Apply/simulate palette on a page](../02-features/apply-simulate-palette.md), [Contrast audit (extension)](../02-features/contrast-audit-extension.md), [Extract page colors](../02-features/extract-page-colors.md), [Web App](web-app.md) (editing hand-off target)

## Objective

A deliberately narrow-scope, fast-utility surface: quick on-page color audits and tests, not a palette editor. Distinct use case from the Web App/plugins — the accessibility angle is explicit (helping someone with a potential visual impairment quickly adapt a page's colors for their own comfort), alongside general "try a palette on a real site before committing" testing.

## Features (three, each its own spec)

1. **Apply/simulate a palette on the current page** — [spec](../02-features/apply-simulate-palette.md). Maps the page's actual colors (literal values or CSS custom properties) to their nearest equivalent in an already-created palette, and applies the result live.
2. **Contrast audit of the current page** — [spec](../02-features/contrast-audit-extension.md). Scores every background/foreground pair on the page against WCAG/APCA, reusing the same contrast math already computed for palette shades.
3. **Extract colors from the current page into a new palette** — [spec](../02-features/extract-page-colors.md). Samples the page's dominant colors as a starting point for a new palette — which can then optionally be applied straight back (feature 1) or refined further.

## Why this hands off to the Web App

The scope is intentionally kept small — this is not where a palette gets built or fine-tuned. Once a palette is extracted here, or a user wants to properly edit/refine a palette before applying it, the extension is expected to deep-link into the [Web App](web-app.md) (most likely via the [Sharing links](../02-features/sharing-links.md) data-in-URL mechanism, so no account or server round-trip is needed just to hand a palette over) rather than duplicating the editor.

## Anchor points to confirm once implementation starts

- **Manifest V3**, content script injected into the active tab (`activeTab`/`scripting` permissions), popup UI for triggering the three actions.
- Likely reuses `engine-ui-color-palette` client-side for color-distance matching (feature 1) and contrast scoring (feature 2) — both are pure, network-free classes already, which fits a content-script bundle; bundle-size impact of pulling in the full engine package is unverified.
- **Storage**: no design-tool document and (unlike the Web App) not necessarily the same browser storage either — needs its own extension-local storage (`chrome.storage`) for extracted palettes and any "currently applied palette per site/tab" state.
- **Account**: not yet decided whether any of the three features require sign-in, or whether they follow the same account-free-to-use philosophy as the Web App (see Open questions).

## Out of scope

- A full palette editor inside the extension — that's the Web App's job (see hand-off above).
- Design-tool sync of any kind — this platform never talks to Figma/Penpot/Sketch/Framer.
- Publishing/subscribing from inside the extension — assumed to require jumping to the Web App/account, not built natively here (see Open questions).

## Open questions

- Does any of the three features require an account, or are all three usable anonymously (at least for a first pass), matching the Web App's "free to try" philosophy?
- Proposed host repo name (`chrome-ui-color-palette`) not confirmed.
- Whether the extension ever needs its own `Platform`/`Editor` enum value, or is only ever tracked as a referrer into the Web App.
- Manifest permissions scope (does contrast audit/extraction need `<all_urls>` host permissions, or is `activeTab` sufficient given it's user-triggered per tab?).

## See also

- [Web App](web-app.md) — the editing surface this platform hands off to
- [Bridge catalog](bridge-actions.md), [Figma](figma.md), [Penpot](penpot.md), [Sketch](sketch.md), [Framer](framer.md) — the design-tool platforms this one has nothing in common with beyond sharing the same engine
- [Palette](../01-domain-model/palette.md) — the shade/contrast model the audit and apply features reuse

## History

| Date | Change |
| --- | --- |
| 2026-08-23 | Created as Draft, consolidated from the requester's description of a narrow-scope Chrome extension (simulate/apply, contrast audit, page color extraction) that hands off to the Web App for editing. No implementation exists yet. |
