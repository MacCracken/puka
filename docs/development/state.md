# puka — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.3.0** — **cut 2026-06-18, awaiting user tag.** Contents: **M3 (framebuffer
renderer + glyphs)** — `src/render/fb.cyr` paints the grid to an in-memory RGB
buffer (full colour model, real CP437 glyphs via **kashi**, the cursor block, and
per-row damage tracking), verified headlessly via a PPM (P6) dump + pixel-asserting
tests and visually (`programs/fb_demo.cyr` → `puka_frame.ppm`). Release gate green:
**261 tests pass**, fmt/lint clean (0 warnings), multi-agent adversarial review
closed (2 findings fixed + regression-tested), VERSION↔cyrius.cyml↔CHANGELOG all
`0.3.0`. The **live on-screen display backend** (push the buffer to a real surface)
is folded into the AGNOS-native bring-up, **M5** — 0.3.0 ships the
platform-agnostic renderer (the Linux fbdev/KMS dev path is a thin adjunct over the
same buffer). (0.2.0 = M2 PTY; 0.1.0 = M1 VT core.) Version map: M(n) → 0.n.0.

## Toolchain

- **Cyrius pin**: `6.2.21` (in `cyrius.cyml [package].cyrius`) — current at scaffold

## Source

- `src/parser.cyr` — VT parser (Williams DEC ANSI state machine). Pure: `vt_feed(byte)` → one typed event.
- `src/grid.cyr` — cell grid (the screen, single source of truth): cells, cursor, scroll region, tabs, scroll/erase/insert/delete primitives.
- `src/unicode.cyr` — UTF-8 decode/encode + `char_width` (wcwidth, UAX#11).
- `src/terminal.cyr` — the driver: parser events → grid mutations (cursor/erase/SGR/scroll/modes/resize) + headless text renderer. Exposes `term_cursor_visible()` (DECTCEM) for the renderer; DECTCEM toggles dirty the cursor row.
- `src/pty.cyr` — PTY + process plumbing (Linux): open/spawn/pump/write/winsize/wait/close. Linux-guarded; agnos backend is M5.
- `src/render/fb.cyr` — framebuffer renderer (M3): grid → RGB pixel buffer. Pure read of the grid; colour resolution (default/16/256/truecolor + bold/dim/reverse/hidden), background paint, kashi glyph blit (VGA 8×16), cursor block, per-row damage consumption, PPM (P6) dump. Integer-only, every pixel write bounds-clamped.
- `src/main.cyr` — demo entry: drives a canned stream through the full pipe, prints the rendered grid.
- `programs/pty_demo.cyr` — live demo: runs `/bin/ls /` inside a PTY, re-renders the captured grid.
- `programs/fb_demo.cyr` — framebuffer demo: drives a styled grid through the pipe and writes `puka_frame.ppm` (the first visible render).

`src/grid.cyr` additionally owns the **per-row damage bitset** (`grid_dirty_rows`),
marked at every write chokepoint and consumed by the renderer
(`grid_row_dirty` / `grid_clear_row_dirty` / `grid_mark_row_dirty` / `grid_mark_all_dirty`).

## Tests

- `tests/parser.tcyr` (70), `tests/grid.tcyr` (66, incl. resize + per-row damage), `tests/unicode.tcyr` (28), `tests/terminal.tcyr` (49 end-to-end), `tests/render.tcyr` (44 — palette/decode/resolve/paint/glyph/cursor/PPM), `tests/pty.tcyr` (2, real fork/exec through a PTY; skip-clean), `tests/puka.tcyr` (2 smoke) — **261 assertions, all green** (`cyrius test`).
- `tests/puka.bcyr` / `tests/puka.fcyr` — bench / fuzz stubs (fuzzing the parser against adversarial input is the M-hardening target).

## Carry-forward / known

- **Large static data warning** (~240KB): grid backing store (~142KB) + kashi's font BSS (~99KB — its glyph tables are u64-unit byte arrays). The renderer's pixel buffer is **heap-allocated** (`alloc`), not static. Acceptable; heap-allocating the grid remains a deferred optimization.
- **Wide CJK glyphs render blank**: kashi's built-in fonts cover CP437 (0x20..0xFF) only, so a width-2 cell paints its background but no glyph until a wider font (PSF/runtime-loaded or `rekha`) lands. The grid/width handling is already correct.
- Deferred (later milestones): live on-screen display backend (open M3 item, below); DA/DSR query responses, charset designators (ESC ( B), alt-screen (1049); mouse/bracketed-paste modes are M6.

## Dependencies

Direct (declared in `cyrius.cyml`):

- stdlib — string, fmt, alloc, io, vec, str, syscalls, assert, bench
- **kashi** — path dep (`../kashi`), **freestanding** `src/font_data.cyr` core only (zero stdlib), exactly as the agnos kernel consumes it. Bitmap console glyphs: built-in CP437 VGA 8×16 / CGA 8×8 / VGA 9×16. v1.0.1, API frozen.

Planned (own-the-stack, not yet wired): `rekha`+`sadish` (vector glyphs, post-v1.0), `mabda`/`ai-hwaccel` (GPU, M6), `sakshi` (logging). kashi's PSF/BDF/PCF loader library face is available but not pulled in (built-in fonts suffice).

## Dep gaps / blockers

- **AGNOS PTY syscall surface** — kernel-side gap; needed for M5 native bring-up (analogous to the `net.cyr` / `vani` agnos-backend gaps).
- **macOS** — needs a Cyrius Darwin backend; out of scope for this repo.

## Consumers

_None yet._ Phase-2 command center (post-v1.0) will be the first.

## Next

**M4 (→ 0.4.0) — input encoding:** keyboard → escape sequences (cursor/function
keys, modifiers, bracketed paste, application-cursor-key mode), wired to
`pty_write` — headless-testable (feed keystrokes → child echoes → grid updates).

The **live on-screen display backend** (Linux fbdev `/dev/fb0` mmap — the closest
analogue to AGNOS `blit` — and/or KMS/DRM for the dev host) drops in as a thin blit
over the existing `fb_buf()`/`fb_width()`/`fb_height()` during **M5**, shared with
the AGNOS `blit`#39 native path; it is *not* a rewrite. Pull it earlier if an
on-screen Linux session is wanted before AGNOS bring-up. See [`roadmap.md`](roadmap.md).

The grid backing store stays fixed-max (resize is a dim-change), so heap-
allocating it remains a deferred optimization, not a blocker.
