# Feature — Palette readability report

- **Status**: Implemented (Claude plugin 1.0.9)
- **Package(s) concerned**: `claude-ui-color-palette`, `skills-ui-color-palette` (skill `ui-color-palette-report-palette`)
- **Related spec**: [Claude](../03-platform-bridges/claude.md), [Preview](preview.md) (same scores and readability formula), [Palette](../01-domain-model/palette.md) §1 (`textContrast`)

## Problem

In Claude, a generated palette could only be seen as an inline swatch table or as the `preview_palette` image (an SVG served by the API). Neither shows the scores, so a designer could not review the readability of a palette, share it with a team, or keep a record of it without opening a design tool. The plugins have a contrast report per shade; Claude had no equivalent and no palette-wide synthesis.

## User flow

1. The user builds or retrieves a palette in Claude (`scale-palette`, `manage-palettes`), or runs an audit.
2. Claude asks how to show the palette: **Readability report** (default), **Quick preview**, or **Skip**. The user can also ask for it directly with `/ui-color-palette:report-palette`.
3. Claude renders one HTML document from the palette rows already in context, without a new API call.
4. If the host can publish pages (an artifact tool), Claude publishes the document and gives the link; otherwise it writes the file in the working directory and gives its path.
5. Claude recaps in a few lines: verdict, global contrast score, WCAG and APCA pass rates, weakest color, one next step.

## Rules

- **One document, all themes.** The report covers every theme passed to `get_palette`, grouped by theme then by color, one row per shade.
- **Per shade**: swatch (rendered with its best text color), hex, value in the palette's color space, WCAG ratio and level for the light and the dark text color, APCA Lc and recommended usage for both, readability score at weight 400 and 700, best text color, and contrast of the shade against the theme background.
- **Color-space column follows the palette's model.** OKLCH, LCH, Lab, OKLab, HSL, HSV, RGB and CMYK are shown natively. HSLuv, P3 and HEX are shown in OKLCH, and the document says so. Values are derived from the hex with the engine's formulas (chroma-js, D65 for Lab/LCH), so they match the plugins to rounding; a source color whose RGB is not on the 8-bit grid can differ by a few hundredths.
- **Contrast is never recomputed.** WCAG ratios, APCA Lc and recommended usage come from the API rows. Only derived labels are computed.
- **WCAG level comes from the ratio**, not from the API's `score` field (which labels everything under 4.5:1 as `A`): AAA ≥ 7, AA ≥ 4.5, AA large ≥ 3, Fail below. WCAG 2.2 is cited; its contrast criteria (1.4.3, 1.4.6, 1.4.11) are those of 2.1.
- **Readability** uses the [Preview](preview.md) formula: |Lc| rescaled to 100 (Lc 108 = 100 %), × 0.85 under weight 400, × 1.15 over 500, capped at 100 %.
- **Global synthesis** reuses the audit scoring so a report and an audit agree: a shade passes WCAG when one text color reaches 4.5:1 and passes APCA when one reaches Lc 60; the global contrast score is the average of both pass rates. Verdict: Excellent ≥ 90 %, Good ≥ 75 %, Fair ≥ 50 %, Poor below. Source colors are listed and marked, never counted.
- **Synthesis content**: verdict, global contrast score, WCAG AA rate, APCA Lc 60 rate, readability index (mean best readability at weight 400), shades reaching AAA, shades unusable for text (< 3:1 with both text colors), strongest and weakest color, mid-tones that pass WCAG but not APCA, safe text color per shade range, and recommendations when an audit provided them.
- **APCA is guidance.** The document states that APCA is the method drafted for WCAG 3 and not a W3C standard.
- **Self-contained.** No script, no external resource: the file opens offline, follows the system light/dark theme, and prints.
- **No minimum font size table.** The APCA font lookup table is © Myndex and is not embedded; the API's recommended usage carries that information.

## Acceptance criteria

- [ ] Given a palette in context, when the user picks the readability report, then no new `get_palette` call is made.
- [ ] Given an OKLCH palette, when the report is rendered, then the color-space column shows OKLCH values equal to the engine's (chroma-js) values to the displayed precision.
- [ ] Given an HSLuv palette, when the report is rendered, then the column shows OKLCH and the header says so.
- [ ] Given a shade with a 3.96:1 ratio for light text, when it is rendered, then its level reads "AA large", not "A".
- [ ] Given the same palette, when it is audited and reported, then both give the same global contrast score.
- [ ] Given a host with an artifact tool, when the report is generated, then it is published and a link is returned; otherwise a local file path is returned.
- [ ] Given a palette whose rows were saved to a file by the host, when the report is generated, then the rows are read from that file and not copied into the conversation.
- [ ] Given `python3` is missing, when the report is requested, then Claude says so and falls back to the inline preview.

## Out of scope

- Editing the palette from the document.
- Vision-deficiency simulation in the report.
- Exporting the report as PDF or Word (the HTML prints to PDF).
- The Figma/Penpot/Sketch/Framer canvas preview boards, which stay as they are.

## Implementation notes

- **Skill**: `ui-color-palette-report-palette` (source of truth in `skills-ui-color-palette`, mirrored in the plugin). Rendering is done by `scripts/render_report.py`, Python 3 standard library only, which also prints a JSON summary used for the recap.
- **Data contract**: `{ palette: { name, description, colorSpace, preset, algorithmVersion }, themes: [{ name, paletteBackground, textColors: { light, dark } }], rows: <compact get_palette rows | path>, synthesis?: { headline, recommendations[] } }`.
- **API / MCP**: reuses `get_palette` with `compact: true`. No new endpoint or tool.
- **Platforms**: Claude only. Design tools keep their contrast report.
- **Analytics**: none (the Claude plugin has no analytics).

## Locales

English only, like the rest of the Claude plugin. No Tolgee keys.

## See also

- [Claude](../03-platform-bridges/claude.md) — the platform this feature lives on
- [Preview](preview.md) — the plugins' per-shade contrast report and readability formula
- [MCP tools](../04-contracts/mcp-tools.md) — `get_palette`

## History

| Date | Change |
| ---- | ------ |
| 2026-11-16 | Created with the Claude plugin 1.0.9 |
