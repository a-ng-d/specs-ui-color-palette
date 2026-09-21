# Feature — Contrast audit of a page (Chrome Extension)

- **Status**: Draft
- **Package(s) concerned**: proposed `chrome-ui-color-palette` (not implemented yet), `engine-ui-color-palette` (`Contrast` class, reused client-side for this use case)
- **UI module**: Chrome Extension only
- **Related spec**: [Chrome Extension](../03-platform-bridges/chrome-extension.md), [Apply/simulate palette on a page](apply-simulate-palette.md) (a plausible one-click fix for a failing pair), [Palette](../01-domain-model/palette.md) §1 (the same `textContrast.wcag`/`textContrast.apca` computation this reapplies to arbitrary page colors instead of palette shades)

## Problem

Checking whether a live website's actual foreground/background color pairs meet accessibility contrast requirements today means manually sampling colors and running them through an external tool. There's no quick, in-context way to audit a real page.

## User flow

1. From the extension's popup, the user triggers "Audit contrast" on the current tab.
2. The extension walks the page's rendered elements, pairing each text node's foreground color with its effective background color.
3. For each pair, a contrast score is computed — reusing the same WCAG + APCA math already used to compute a palette shade's `textContrast` (see [Palette](../01-domain-model/palette.md) §1).
4. Results are surfaced as a list and/or an on-page overlay: which pairs pass/fail, at which level (WCAG AA/AAA, or the relevant APCA threshold), with failing elements highlighted directly on the page.

## Rules

- **Same contrast math as the rest of the product** — this reuses the engine's existing WCAG/APCA computation rather than introducing a second scoring method, so results here are consistent with what the palette editor already reports for its own shades.
- **Effective background resolution** needs to walk up the DOM to account for transparency and inherited/ancestor backgrounds — a text node rendered over a semi-transparent element needs its background flattened through the ancestor chain, not just read off its immediate parent. Exact algorithm not decided (see Open questions).
- **Report-only** in this feature — it flags failing pairs, it does not change them (that's [Apply/simulate](apply-simulate-palette.md)'s job; whether the two are wired together for a one-click fix is open).

## Acceptance criteria

- [ ] Given a page audited, then every text/background pair on the page gets a computed WCAG and APCA score.
- [ ] Given a pair that fails the relevant threshold, then it's flagged as failing and highlighted on the page.
- [ ] Given a pair with a semi-transparent background over other content, then the effective background used for scoring reflects the flattened/composited color, not just the immediate parent's raw background.
- [ ] Given an audit result, then no colors on the page are changed as a result of running this feature alone.

## Out of scope

- Fixing failing pairs — this feature only reports; see [Apply/simulate](apply-simulate-palette.md) for the mechanism that could plausibly fix one.
- Non-text contrast requirements (e.g. UI component/icon contrast per WCAG 2.2's non-text criteria) — not mentioned by the requester, assumed text-only for this draft (see Open questions).

## Implementation notes

- **Content script**: DOM walk pairing each text node with its resolved foreground/effective background.
- **Engine reuse**: `engine-ui-color-palette`'s `Contrast` class, the same one backing `textContrast.wcag`/`textContrast.apca` on palette shades (see [Palette](../01-domain-model/palette.md) §1) — applied here to arbitrary sampled page colors instead of generated shades.
- **Overlay UI**: highlighting failing elements directly on the page implies injecting some overlay/outline styling — needs to avoid interfering with the audited page's own layout.

## Locales

| Text | Where | Tolgee key | Status |
| --- | --- | --- | --- |
| "Audit contrast" action label, pass/fail result copy | Extension popup / on-page overlay | — | Unread — no mockup yet |

## See also

- [Chrome Extension](../03-platform-bridges/chrome-extension.md) — parent platform doc
- [Apply/simulate palette on a page](apply-simulate-palette.md) — plausible fix mechanism for a failing pair
- [Palette](../01-domain-model/palette.md) §1 — the contrast computation this reuses

## Open questions

- Exact effective-background flattening algorithm through the ancestor chain.
- Whether non-text contrast (UI components, icons) is ever in scope.
- Whether a failing pair offers a direct "apply nearest palette color to fix this" action, bridging into [Apply/simulate](apply-simulate-palette.md).
- Whether this feature requires sign-in (see [Chrome Extension](../03-platform-bridges/chrome-extension.md)'s Open questions).

## History

| Date | Change |
| --- | --- |
| 2026-08-23 | Created as Draft, consolidated from the requester's description of auditing a live page's background/foreground contrast scores. No implementation exists yet. |
