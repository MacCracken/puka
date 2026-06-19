# puka — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.5.0** — **cut 2026-06-18, awaiting user tag.** Contents: **M5 (Linux live
terminal)** — the live edges over the M1–M4 core: `src/render/fbdev.cyr` (blit the
RGB buffer to `/dev/fb0`, generic over 16/24/32-bpp truecolor layouts),
`src/input/evdev.cyr` (decode raw `/dev/input` scancodes → `input_encode`, US
keymap + independent L/R modifiers), and `programs/puka_session.cyr` (the
single-threaded interactive loop hosting `/bin/sh`). puka now runs as a real
interactive terminal on a Linux framebuffer/VT. Release gate green: **410 tests
pass**, fmt/lint clean (0 warnings), multi-agent adversarial review closed (4
fixes applied + regression-tested), the fbdev ABI offsets validated read-only
against real hardware (2560×1440 XRGB8888), VERSION↔cyrius.cyml↔CHANGELOG all
`0.5.0`. The pure cores are headless-tested; the device layers are Linux-guarded;
the live on-screen session runs on a bare Linux VT (user-side — it can't run under
a compositor or in CI). **AGNOS-native bring-up is post-v1.0.** (0.4.0 = M4 input;
0.3.0 = M3 renderer; 0.2.0 = M2 PTY; 0.1.0 = M1 VT core.) Version map: M(n) → 0.n.0.

## Toolchain

- **Cyrius pin**: `6.2.21` (in `cyrius.cyml [package].cyrius`) — current at scaffold

## Source

- `src/parser.cyr` — VT parser (Williams DEC ANSI state machine). Pure: `vt_feed(byte)` → one typed event.
- `src/grid.cyr` — cell grid (the screen, single source of truth): cells, cursor, scroll region, tabs, scroll/erase/insert/delete primitives.
- `src/unicode.cyr` — UTF-8 decode/encode + `char_width` (wcwidth, UAX#11).
- `src/terminal.cyr` — the driver: parser events → grid mutations (cursor/erase/SGR/scroll/modes/resize) + headless text renderer. Exposes `term_cursor_visible()` (DECTCEM, dirties the cursor row on toggle), `term_app_cursor_get()` (DECCKM) and `term_bracket_paste_get()` (mode 2004) for the renderer/encoder.
- `src/pty.cyr` — PTY + process plumbing (Linux): open/spawn/pump/write/winsize/wait/close. Linux-guarded; the agnos backend is post-v1.0.
- `src/render/fb.cyr` — framebuffer renderer (M3): grid → RGB pixel buffer. Pure read of the grid; colour resolution (default/16/256/truecolor + bold/dim/reverse/hidden), background paint, kashi glyph blit (VGA 8×16), cursor block, per-row damage consumption, PPM (P6) dump. Integer-only, every pixel write bounds-clamped.
- `src/input.cyr` — keyboard→escape-sequence encoder (M4): `input_encode(sym, mods, out)` (disjoint keysym range; xterm modifier formula via `input__xtmod`) + `input_paste(text, len, cap, out)` (bracketed-paste 2004 wrap + ESC/0x9B strip + cap bound). Pure; reads terminal modes via getters.
- `src/render/fbdev.cyr` — Linux `/dev/fb0` display backend (M5): pure RGB→device-format pixel pack + bounds-clamped blit (16/24/32-bpp, stride-aware) over `fb_buf()`; Linux-guarded open/ioctl(screeninfo)/mmap/present, with device geometry validated before the blit trusts it.
- `src/input/evdev.cyr` — Linux `/dev/input/eventN` keyboard source (M5): pure `input_event` decode + US-QWERTY keymap + independent L/R modifier tracking → `input_encode`; Linux-guarded open/read/`EVIOCGRAB`. Untrusted-device-bounded.
- `src/main.cyr` — demo entry: drives a canned stream through the full pipe, prints the rendered grid.
- `programs/pty_demo.cyr` — live demo: runs `/bin/ls /` inside a PTY, re-renders the captured grid.
- `programs/fb_demo.cyr` — framebuffer demo: drives a styled grid through the pipe and writes `puka_frame.ppm` (the first visible render).
- `programs/input_demo.cyr` — input demo: "types" into `/bin/cat` via `input_encode` → `pty_write`, pumps the echo, re-renders (the input path end to end).
- `programs/puka_session.cyr` — **M5 capstone**: the live interactive terminal — spawns `/bin/sh`, busy-polls evdev + the PTY master, encodes keys → child, pumps output → grid, renders → `/dev/fb0`. Run on a bare Linux VT.

`src/grid.cyr` additionally owns the **per-row damage bitset** (`grid_dirty_rows`),
marked at every write chokepoint and consumed by the renderer
(`grid_row_dirty` / `grid_clear_row_dirty` / `grid_mark_row_dirty` / `grid_mark_all_dirty`).

## Tests

- `tests/parser.tcyr` (70), `tests/grid.tcyr` (66, resize + per-row damage), `tests/unicode.tcyr` (28), `tests/terminal.tcyr` (55, + DECCKM/2004 getters), `tests/render.tcyr` (44 — palette/resolve/paint/glyph/cursor/PPM), `tests/input.tcyr` (67 — byte-exact + `vt_feed` round-trips + paste), `tests/fbdev.tcyr` (25 — pixel pack + stride/clamp blit vs a fake fb), `tests/evdev.tcyr` (49 — synthetic-event decode + L/R modifiers), `tests/pty.tcyr` (2) + `tests/input_pty.tcyr` (2, real PTY echo — both skip-clean), `tests/puka.tcyr` (2 smoke) — **410 assertions, all green** (`cyrius test`).
- `tests/puka.bcyr` / `tests/puka.fcyr` — bench / fuzz stubs (fuzzing the parser against adversarial input is the M-hardening target).

## Carry-forward / known

- **Large static data warning** (~240KB): grid backing store (~142KB) + kashi's font BSS (~99KB — its glyph tables are u64-unit byte arrays). The renderer's pixel buffer is **heap-allocated** (`alloc`), not static. Acceptable; heap-allocating the grid remains a deferred optimization.
- **Wide CJK glyphs render blank**: kashi's built-in fonts cover CP437 (0x20..0xFF) only, so a width-2 cell paints its background but no glyph until a wider font (PSF/runtime-loaded or `rekha`) lands. The grid/width handling is already correct.
- Deferred (M6 conformance / post-v1.0): DA/DSR query responses, charset designators (ESC ( B), alt-screen (1049), scrollback ring, origin-mode edge cases, mouse tracking, grapheme clustering; non-US keymaps + CapsLock in evdev; AGNOS-native edges (display/input/PTY) are post-v1.0.

## Dependencies

Direct (declared in `cyrius.cyml`):

- stdlib — string, fmt, alloc, io, vec, str, syscalls, assert, bench
- **kashi** — pinned **git dep** (`git`/`tag = "1.0.1"`), **freestanding** `src/font_data.cyr` core only (zero stdlib) — the same core the agnos kernel consumes. Resolves identically on a devbox and in CI (no sibling checkout). Bitmap console glyphs: built-in CP437 VGA 8×16 / CGA 8×8 / VGA 9×16. v1.0.1, API frozen.

Planned (own-the-stack, not yet wired): `rekha`+`sadish` (vector glyphs, post-v1.0), `mabda`/`ai-hwaccel` (GPU, M6), `sakshi` (logging). kashi's PSF/BDF/PCF loader library face is available but not pulled in (built-in fonts suffice).

## Dep gaps / blockers

- **AGNOS framebuffer-console environment** — AGNOS can't yet host puka (no console/display environment), so **AGNOS-native bring-up is post-v1.0**; v1.0 is Linux-complete. The kernel-side **AGNOS PTY syscall surface** (analogous to the `net.cyr` / `vani` agnos-backend gaps) is part of that post-v1.0 work, not a v1.0 blocker.
- **macOS** — needs a Cyrius Darwin backend; out of scope for this repo.

## Consumers

_None yet._ Phase-2 command center (post-v1.0) will be the first.

**M6 (→ 0.6.0 … 0.9.0) — conformance + polish.** With the full Linux terminal
working end to end (M1–M5), M6 makes it *correct and complete*: a curated vttest /
`ctlseqs` conformance pass; **alternate screen** (1049); scrollback ring; origin
mode; charset switching (DEC special graphics); DA/DSR query responses (now that
there's a host writer); mouse tracking (SGR); grapheme clustering; and **fuzzing the
parser hard** against malformed/truncated/adversarial input (the untrusted-input
boundary). Then **M7** is the v1.0 hardening sweep + engine-API freeze.

**AGNOS-native bring-up stays post-v1.0** (AGNOS lacks a console environment to host
puka). v1.0 is a Linux-complete terminal; the AGNOS edges (`blit`#39 + xHCI/HID +
kernel PTY) drop in behind the same seams when AGNOS is ready. See
[`roadmap.md`](roadmap.md).

The grid backing store stays fixed-max (resize is a dim-change), so heap-
allocating it remains a deferred optimization, not a blocker.
