# Feature — Apply/simulate a palette on a page (Chrome Extension)

- **Status**: Draft
- **Package(s) concerned**: proposed `chrome-ui-color-palette` (not implemented yet, see [Chrome Extension](../03-platform-bridges/chrome-extension.md)), `engine-ui-color-palette` (color-distance/nearest-match logic, reused client-side, not wired for this use case yet)
- **UI module**: Chrome Extension only — no equivalent in the plugins or Web App
- **Related spec**: [Chrome Extension](../03-platform-bridges/chrome-extension.md), [Contrast audit (extension)](contrast-audit-extension.md) (a natural follow-up: fix a failing pair by applying the nearest palette color), [Extract page colors](extract-page-colors.md) (a natural precursor: try applying the palette you just extracted from the same page)

## Problem

Someone who already has a palette — built for accessibility reasons, brand testing, or personal preference — has no quick way to see it actually applied across a real website, or to make that change stick for their own browsing comfort, without a designer/developer touching the site's code. The requester's use case is explicitly accessibility-first: helping someone with a potential visual/color-vision impairment adapt a page's colors for themselves.

## User flow

1. From the extension's popup, the user picks one of their existing palettes (own local, published, or — pending [Team spaces](team-spaces.md) — a team one).
2. The user triggers "Simulate"/"Apply" against the current tab.
3. The extension scans the active page's currently-rendered colors — both colors declared as CSS custom properties and colors that are hardcoded/literal — and maps each one to its nearest match in the selected palette.
4. Matches are applied live on the page:
   - a color declared via a CSS custom property has that property's value swapped once at its declaration, letting the existing cascade propagate the change;
   - a hardcoded/literal color has no single declaration to change, so it's overridden at each place it's computed to apply.
5. The user can revert to the page's original colors at any time.

## Rules

- **Two distinct replacement mechanisms**, chosen per color depending on how it's authored on the page — a custom-property swap (cheap, cascades naturally) vs. a per-occurrence literal override (more invasive, potentially many injected overrides on a color-heavy page).
- **"Nearest match" needs a defined, perceptual distance metric** — expected to reuse the engine's own color-space handling (e.g. OKLCH/LAB-based distance) rather than naive RGB distance, for consistency with how the rest of the product reasons about color, but the exact metric and any "too far to match" threshold are not decided (see Open questions).
- **Session-scoped, never written back to the site** — this is a personal overlay on top of what the user sees, not a change proposal for the site owner.

## Acceptance criteria

- [ ] Given a palette selected in the extension popup and a current tab, when "Apply" is triggered, then every distinct rendered color on the page is mapped to its nearest palette color.
- [ ] Given a page color declared via a CSS custom property, when applied, then only the property's declared value is changed (not every individual usage).
- [ ] Given a page color that is hardcoded/literal, when applied, then every occurrence where that literal color is computed is overridden.
- [ ] Given an applied palette, when the user reverts, then the page returns to its original, unmodified colors.
- [ ] Given the applied state, then nothing is sent back to the site or persisted server-side — the change only affects the current viewer's browser.

## Out of scope

- Images, gradients, and raster/video content — color matching applies to solid, styleable colors only; whether gradients are addressed at all is open (see Open questions).
- Persisting the applied palette permanently for the site owner — this is a viewer-side accessibility/testing overlay, not a publishing mechanism.
- Building/editing the palette itself inside the extension — see [Chrome Extension](../03-platform-bridges/chrome-extension.md)'s hand-off to the Web App.

## Implementation notes

- **Content script**: scans computed styles across the page's DOM/CSSOM to enumerate rendered colors, distinguishing custom-property-sourced values from literals.
- **Engine reuse**: nearest-match logic is expected to reuse `engine-ui-color-palette`'s existing color classes (`Color`) rather than reimplementing color-distance math — bundle-size impact of pulling the engine into a content script is unverified (see [Chrome Extension](../03-platform-bridges/chrome-extension.md)'s Open questions).
- **Storage**: which palette is currently applied per site/tab needs its own `chrome.storage` entry so a revert is possible and (open question) so the applied state can optionally persist across reloads.

## Locales

| Text | Where | Tolgee key | Status |
| --- | --- | --- | --- |
| "Simulate" / "Apply" / "Revert" action labels | Extension popup | — | Unread — no mockup yet |

## See also

- [Chrome Extension](../03-platform-bridges/chrome-extension.md) — parent platform doc
- [Contrast audit (extension)](contrast-audit-extension.md) — a plausible one-click fix target once a failing pair is identified there
- [Extract page colors](extract-page-colors.md) — a plausible source palette for this feature, extracted from the very page being tested

## Open questions

- Exact perceptual distance metric and "too far to match" fallback behavior (force closest anyway vs. leave untouched).
- Whether gradients/images are addressed at all, even partially.
- Whether there should be a cap on the number of literal-color overrides injected per page, given that mechanism doesn't scale as cleanly as a property swap.
- Whether the applied state persists across reloads/navigation within the same site, or resets every time.
- Whether this feature requires sign-in (see [Chrome Extension](../03-platform-bridges/chrome-extension.md)'s Open questions).

## History

| Date | Change |
| --- | --- |
| 2026-08-23 | Created as Draft, consolidated from the requester's description of applying a palette's nearest colors to a live page's hardcoded/property-based colors, with an explicit accessibility use case. No implementation exists yet. |
