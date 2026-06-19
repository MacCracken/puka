# puka — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.4.0** — **cut 2026-06-18, awaiting user tag.** Contents: **M4 (input
encoding)** — `src/input.cyr`, a pure keyboard→escape-sequence encoder
(`input_encode` / `input_paste`), byte-exact against xterm `ctlseqs` and
round-tripped through puka's own parser: printable/control/alt, cursor & nav &
function keys, the xterm modifier formula, DECCKM application-cursor mode, and
bracketed paste (mode 2004) with anti-injection sanitization. Release gate green:
**336 tests pass**, fmt/lint clean (0 warnings), multi-agent adversarial review
closed (1 low-sev hardening applied + regression-tested), VERSION↔cyrius.cyml↔
CHANGELOG all `0.4.0`. The **live raw-key source** (Linux evdev / AGNOS xHCI-HID)
+ interactive loop ride with the **M5** display backend — 0.4.0 ships the
platform-agnostic encoder + a proven PTY round-trip (encoded keys → child → echo →
grid). (0.3.0 = M3 renderer; 0.2.0 = M2 PTY; 0.1.0 = M1 VT core.) Version map:
M(n) → 0.n.0.

## Toolchain

- **Cyrius pin**: `6.2.21` (in `cyrius.cyml [package].cyrius`) — current at scaffold

## Source

- `src/parser.cyr` — VT parser (Williams DEC ANSI state machine). Pure: `vt_feed(byte)` → one typed event.
- `src/grid.cyr` — cell grid (the screen, single source of truth): cells, cursor, scroll region, tabs, scroll/erase/insert/delete primitives.
- `src/unicode.cyr` — UTF-8 decode/encode + `char_width` (wcwidth, UAX#11).
- `src/terminal.cyr` — the driver: parser events → grid mutations (cursor/erase/SGR/scroll/modes/resize) + headless text renderer. Exposes `term_cursor_visible()` (DECTCEM, dirties the cursor row on toggle), `term_app_cursor_get()` (DECCKM) and `term_bracket_paste_get()` (mode 2004) for the renderer/encoder.
- `src/pty.cyr` — PTY + process plumbing (Linux): open/spawn/pump/write/winsize/wait/close. Linux-guarded; agnos backend is M5.
- `src/render/fb.cyr` — framebuffer renderer (M3): grid → RGB pixel buffer. Pure read of the grid; colour resolution (default/16/256/truecolor + bold/dim/reverse/hidden), background paint, kashi glyph blit (VGA 8×16), cursor block, per-row damage consumption, PPM (P6) dump. Integer-only, every pixel write bounds-clamped.
- `src/input.cyr` — keyboard→escape-sequence encoder (M4): `input_encode(sym, mods, out)` (disjoint keysym range; xterm modifier formula via `input__xtmod`) + `input_paste(text, len, cap, out)` (bracketed-paste 2004 wrap + ESC/0x9B strip + cap bound). Pure; reads terminal modes via getters.
- `src/main.cyr` — demo entry: drives a canned stream through the full pipe, prints the rendered grid.
- `programs/pty_demo.cyr` — live demo: runs `/bin/ls /` inside a PTY, re-renders the captured grid.
- `programs/fb_demo.cyr` — framebuffer demo: drives a styled grid through the pipe and writes `puka_frame.ppm` (the first visible render).
- `programs/input_demo.cyr` — input demo: "types" into `/bin/cat` via `input_encode` → `pty_write`, pumps the echo, re-renders (the input path end to end).

`src/grid.cyr` additionally owns the **per-row damage bitset** (`grid_dirty_rows`),
marked at every write chokepoint and consumed by the renderer
(`grid_row_dirty` / `grid_clear_row_dirty` / `grid_mark_row_dirty` / `grid_mark_all_dirty`).

## Tests

- `tests/parser.tcyr` (70), `tests/grid.tcyr` (66, resize + per-row damage), `tests/unicode.tcyr` (28), `tests/terminal.tcyr` (55, + DECCKM/2004 getters), `tests/render.tcyr` (44 — palette/decode/resolve/paint/glyph/cursor/PPM), `tests/input.tcyr` (67 — byte-exact + `vt_feed` round-trips + paste), `tests/pty.tcyr` (2) + `tests/input_pty.tcyr` (2, real PTY echo — both skip-clean), `tests/puka.tcyr` (2 smoke) — **336 assertions, all green** (`cyrius test`).
- `tests/puka.bcyr` / `tests/puka.fcyr` — bench / fuzz stubs (fuzzing the parser against adversarial input is the M-hardening target).

## Carry-forward / known

- **Large static data warning** (~240KB): grid backing store (~142KB) + kashi's font BSS (~99KB — its glyph tables are u64-unit byte arrays). The renderer's pixel buffer is **heap-allocated** (`alloc`), not static. Acceptable; heap-allocating the grid remains a deferred optimization.
- **Wide CJK glyphs render blank**: kashi's built-in fonts cover CP437 (0x20..0xFF) only, so a width-2 cell paints its background but no glyph until a wider font (PSF/runtime-loaded or `rekha`) lands. The grid/width handling is already correct.
- Deferred (later milestones): live on-screen display backend (open M3 item, below); DA/DSR query responses, charset designators (ESC ( B), alt-screen (1049); mouse/bracketed-paste modes are M6.

## Dependencies

Direct (declared in `cyrius.cyml`):

- stdlib — string, fmt, alloc, io, vec, str, syscalls, assert, bench
- **kashi** — pinned **git dep** (`git`/`tag = "1.0.1"`), **freestanding** `src/font_data.cyr` core only (zero stdlib) — the same core the agnos kernel consumes. Resolves identically on a devbox and in CI (no sibling checkout). Bitmap console glyphs: built-in CP437 VGA 8×16 / CGA 8×8 / VGA 9×16. v1.0.1, API frozen.

Planned (own-the-stack, not yet wired): `rekha`+`sadish` (vector glyphs, post-v1.0), `mabda`/`ai-hwaccel` (GPU, M6), `sakshi` (logging). kashi's PSF/BDF/PCF loader library face is available but not pulled in (built-in fonts suffice).

## Dep gaps / blockers

- **AGNOS PTY syscall surface** — kernel-side gap; needed for M5 native bring-up (analogous to the `net.cyr` / `vani` agnos-backend gaps).
- **macOS** — needs a Cyrius Darwin backend; out of scope for this repo.

## Consumers

_None yet._ Phase-2 command center (post-v1.0) will be the first.

## Next

**M5 (→ 0.5.0) — the live edges (display + key source) + AGNOS-native bring-up.**
The platform-agnostic core is complete through M4 (parse → grid → render → encode),
all headless-testable. M5 wires the two live edges over it:
1. **Display** — push the renderer's pixel buffer to a real surface: Linux fbdev
   (`/dev/fb0` mmap, the closest analogue to AGNOS `blit`) and/or KMS/DRM for the dev
   host; the AGNOS `blit`#39 framebuffer for native. A thin blit over `fb_buf()` /
   `fb_width()` / `fb_height()` — not a rewrite.
2. **Key source** — feed `input_encode(sym, mods, …)` from a real raw-key device:
   Linux evdev (`/dev/input`) on the dev host, the AGNOS xHCI/HID path for native.
3. The **interactive loop** (`pty_pump` ⇄ render ⇄ key-read) + the **AGNOS PTY
   syscall surface**, booting a shell on the AGNOS framebuffer — the proof-app.

Either Linux edge can be pulled forward for an on-screen Linux session before the
AGNOS work. See [`roadmap.md`](roadmap.md).

The grid backing store stays fixed-max (resize is a dim-change), so heap-
allocating it remains a deferred optimization, not a blocker.
