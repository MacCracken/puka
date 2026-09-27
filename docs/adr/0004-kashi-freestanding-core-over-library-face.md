# 0004 — Consume kashi's freestanding font-data core, not its library face

**Status**: Accepted
**Date**: 2026-09-27 (first recorded in `cyrius.cyml` on 2026-08-17; moved here at 0.6.24)

## Context

puka draws its glyphs from kashi's built-in CP437 fonts. `src/render/fb.cyr` and
`src/render/atlas.cyr` call `kashi_font_init`, `kashi_glyph_row`, `kashi_font_width` and
`kashi_font_height` against the VGA 8×16 face.

kashi ships two faces (kashi ADR 0001). The **freestanding core**, `src/font_data.cyr`, holds the
glyph tables and their no-stdlib accessors; it is what the agnos kernel links. The **library
face**, `dist/kashi.cyr`, is the core plus PSF/BDF/PCF import and a runtime font registry. kashi's
ADR says stdlib-linked consumers should take the library face, and puka links the full stdlib, so
the core looks like the wrong surface.

For most of puka's history it was the only surface that could be used:

- Until kashi 1.0.6 the library face could not be consumed downstream. `src/lib.cyr` opens with
  `include "src/font_data.cyr"`, a kashi-root-relative path that fails once vendored:
  `error: cannot open include file: src/font_data.cyr`.
- kashi 1.0.6 added the `dist/kashi.cyr` bundle, but `dist/` stayed out of git until 1.0.10, so a
  consumer resolving kashi by tag could not reach it before 1.0.10.

puka now resolves kashi 1.0.10 by tag, so the choice is real.

## Decision

**puka declares `[deps.kashi] modules = ["src/font_data.cyr"]`, the core only.** It calls nothing
the library face adds.

The cost of the alternative was measured on dhancha, which makes the same choice, on 2026-08-17 at
kashi 1.0.6. The library face cost 548,000 B against the core's 364,640 B, which is **+183,360 B
(+50%)**, and `CYRIUS_DCE=1` reclaimed none of it. This was measured on dhancha, not on puka.

## Consequences

- **Positive**: no bytes spent on font loading that nothing calls. puka reads the same glyph core
  as the agnos kernel and dhancha.
- **Negative**: only the built-in CP437 fonts. A wide (CJK) cell paints its background and no glyph,
  because the Unicode tables exist only for runtime-loaded fonts in the library face.
- **Neutral**: dhancha, which puka pulls in, declares the same core. A switch has to be checked
  against the resolved graph, not only against this manifest.

## Reopen when

puka needs runtime font loading: a PSF/BDF/PCF face for wide or non-CP437 glyphs, or a
user-selectable font. Then switch to `modules = ["dist/kashi.cyr"]` and measure the cost on puka
itself. The vector path (`rekha` + `sadish`) is the other route to wide glyphs; see the roadmap.

## Alternatives considered

- **The library face (`dist/kashi.cyr`).** Declined for now: +50% for code puka does not call.
- **Copy the glyph tables into puka.** Rejected: it inlines what a sibling crate owns (CLAUDE.md,
  *Own the stack*).
