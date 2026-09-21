# Feature — Extract colors from a page (Chrome Extension)

- **Status**: Draft
- **Package(s) concerned**: proposed `chrome-ui-color-palette` (not implemented yet), `engine-ui-color-palette` (`DominantColors` class, reused for this use case), `web-ui-color-palette` (hand-off target for full editing, via [Sharing links](sharing-links.md))
- **UI module**: Chrome Extension only
- **Related spec**: [Chrome Extension](../03-platform-bridges/chrome-extension.md), [Image extraction](image-extraction.md) (the existing feature that already runs the same engine class against an uploaded image), [Apply/simulate palette on a page](apply-simulate-palette.md) (the requester's "pourquoi pas l'appliquer" follow-up), [Sharing links](sharing-links.md) (hand-off mechanism to the Web App)

## Problem

Someone likes a website's color scheme and wants to turn it into a reusable palette, instead of eyeballing and manually re-entering hex codes.

## User flow

1. From the extension's popup, the user triggers "Extract colors" on the current tab.
2. The extension samples the page's dominant/most-used rendered colors.
3. The extracted colors are offered as source colors for a **new** palette.
4. From there the user can either:
   - hand the new palette off to the [Web App](../03-platform-bridges/web-app.md) (via the [Sharing links](sharing-links.md) data-only URL mechanism, no account or server round-trip needed) to properly edit it (scale, themes, export); or
   - immediately try [applying](apply-simulate-palette.md) that same freshly-extracted palette back onto the page it came from.

## Rules

- **Extraction source is the rendered page**, not its raw CSS source — computed styles / a rendered snapshot, so what gets extracted matches what's actually visible, consistent with the extension's overall "audit what's live" philosophy (same principle as [Contrast audit](contrast-audit-extension.md)).
- **Reuses the same dominant-color engine class already used for image extraction** ([Image extraction](image-extraction.md)) rather than a second, bespoke algorithm — though that class's existing input shape is pixel data from an image, not DOM content, so a page-specific input adapter (e.g. rendering the visible viewport to a canvas first) is likely needed rather than a direct reuse (see Open questions).

## Acceptance criteria

- [ ] Given a page, when "Extract colors" is triggered, then a new palette is created seeded with that page's dominant colors.
- [ ] Given an extracted palette, then it can be sent to the Web App via a share link without requiring an account.
- [ ] Given an extracted palette, then it can alternatively be applied directly back to the same page via [Apply/simulate](apply-simulate-palette.md).

## Out of scope

- Full palette editing (scale, themes, export) inside the extension — deliberately deferred to the Web App hand-off.
- Extracting from images/files the user provides directly — that's the existing [Image extraction](image-extraction.md) feature; this one is page-specific.

## Implementation notes

- **Content script / capture**: needs some way to get pixel data for the visible page (e.g. rendering to an off-screen canvas) to feed the existing dominant-color algorithm, since that algorithm expects image pixel data, not DOM nodes.
- **Engine reuse**: `engine-ui-color-palette`'s `DominantColors` class, shared with [Image extraction](image-extraction.md) — exact adapter shape between "rendered page" and "image pixel data" not designed yet.
- **Hand-off**: relies on [Sharing links](sharing-links.md)'s data-only URL shape being implemented first (or in parallel) to be useful.

## Locales

| Text | Where | Tolgee key | Status |
| --- | --- | --- | --- |
| "Extract colors" action label | Extension popup | — | Unread — no mockup yet |

## See also

- [Chrome Extension](../03-platform-bridges/chrome-extension.md) — parent platform doc
- [Image extraction](image-extraction.md) — the sibling feature this reuses the engine class from
- [Apply/simulate palette on a page](apply-simulate-palette.md) — optional immediate next step
- [Sharing links](sharing-links.md) — hand-off mechanism to the Web App

## Open questions

- Exact capture mechanism to turn a rendered page into pixel data for the existing `DominantColors` algorithm.
- How many colors get extracted by default, and whether that's user-configurable (mirrors a likely-relevant open question in [Image extraction](image-extraction.md), not re-litigated here).
- Whether this feature requires sign-in (see [Chrome Extension](../03-platform-bridges/chrome-extension.md)'s Open questions).

## History

| Date | Change |
| --- | --- |
| 2026-08-23 | Created as Draft, consolidated from the requester's description of extracting a page's colors into a new palette, optionally applying it straight back. No implementation exists yet. |
