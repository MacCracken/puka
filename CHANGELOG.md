# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

## [0.3.0] — 2026-06-18

**M3 — framebuffer renderer + glyphs.** puka's first *visible* surface. 0.3.0
ships the platform-agnostic renderer (grid → RGB pixel buffer + glyphs + cursor +
per-row damage), verified headlessly via PPM and visually via `fb_demo`. The live
on-screen display backend — pushing this buffer to a real screen — is folded into
the AGNOS-native bring-up (**M5**); on Linux it is a thin fbdev/KMS adjunct over
the same `fb_buf()`.

### Added
- **`src/render/fb.cyr`** — the renderer. A pure read of the grid (single-source-of-truth invariant: the renderer never mutates) that resolves the full xterm colour model to 24-bit RGB — default fg/bg, the 16 ANSI colours, the 16..231 6×6×6 cube (`level(d)=d?d*40+55:0`), the 232..255 grayscale ramp, and truecolor — applying `bold`=bright / `dim` / `reverse` / `hidden` in the correct order (cursor reuses the reverse swap, so SGR-reverse under the cursor cancels). It paints each cell's background rectangle, blits real CP437 glyphs via **kashi**, draws the cursor block (DECTCEM-aware), tracks per-row damage so a frame repaints only changed rows, and serializes the pixel buffer to a PPM (P6) image — the headless, pixel-assertable verification seam (the renderer's analogue of `term_render_row` for text). Integer-only; every pixel write is bounds-clamped (the renderer is a fresh untrusted-input boundary — cell glyph/colour/attr ultimately come from adversarial PTY output). `programs/fb_demo.cyr` renders a styled grid (SGR colours, bold, reverse, 256-colour, truecolor, cursor) to `puka_frame.ppm`.
- **kashi dependency**: wired the **kashi** crate (v1.0.1, API-frozen) as a **pinned git dep** (`git = ".../kashi.git", tag = "1.0.1"`) — selecting only its **freestanding** `src/font_data.cyr` core (zero stdlib: `store8`/`load8` + arithmetic), the same core the agnos kernel consumes. A pinned git dep (rather than a sibling path dep) resolves identically on a devbox and in CI, so no sibling checkout is needed. Gives the built-in CP437 bitmap fonts (VGA 8×16 / CGA 8×8 / VGA 9×16); the PSF/BDF/PCF loader library face is deliberately *not* pulled in (`modules = ["src/font_data.cyr"]`).
- **Per-row damage tracking** (`grid.cyr`): a `u64` dirty-row bitset marked at every grid write chokepoint — cell writes, row copy/blank, insert/delete, cursor moves (mark **both** old and new rows), `init`/`resize` (mark all) — consumed + cleared by the renderer each frame. Public surface: `grid_row_dirty` / `grid_clear_row_dirty` / `grid_mark_row_dirty` / `grid_mark_all_dirty`. +14 `grid.tcyr` assertions.
- **`term_cursor_visible()`** (`terminal.cyr`): DECTCEM visibility accessor for the renderer; the DECTCEM (`?25h`/`?25l`) handler now dirties the cursor row on a real toggle so the block repaints even on an otherwise-quiescent screen.
- **`tests/render.tcyr`** — 44 assertions: palette/cube/grayscale/clamp, token decode, the attribute resolve order, background + cursor pixel paint, the kashi glyph blit ('A' lights pixels, blank cell does not), the DECTCEM re-dirty fix, the `fb_init` geometry-overflow guard, and the PPM P6 header.

### Reviewed
- Multi-agent adversarial review (correctness / bounds & overflow / Cyrius-idiom / untrusted-input), every finding independently verified before acceptance. Fixed two confirmed defects, both regression-tested: a DECTCEM dropped-repaint (a bare cursor hide/show on a settled screen left a ghost / failed to reappear), and a `fb_init` integer-overflow where an out-of-range cell size could wrap the buffer size to a small positive value and undersize the allocation while the plot bounds stayed huge (now cell metrics are capped and the geometry is overflow-checked before `alloc`).

## [0.2.0] — 2026-06-18

### Added
- **M2 — PTY + process plumbing (Linux)**: `src/pty.cyr` — allocate a pseudo-terminal (`/dev/ptmx` → `TIOCSPTLCK` unlock → `TIOCGPTN` → `/dev/pts/N`), `fork` + child-side controlling-tty setup (`setsid` / `TIOCSCTTY` / `dup2` slave→0,1,2) + `execve` with explicit argv (never a shell string), bounded non-blocking pump (`pty_pump`) that reads the master and feeds bytes into `term_feed`, plus `pty_write` (input path, used in M4), `pty_set_winsize`, `pty_wait`, `pty_close`. Linux-guarded (`CYRIUS_TARGET_LINUX`) with non-Linux stubs; agnos PTY is M5. `tests/pty.tcyr` — spawns a real `/bin/echo` and asserts its output lands in the grid (skip-clean if the sandbox blocks `/dev/ptmx`/fork).
- **Resize**: `grid_resize` + `term_resize` — logical screen resize (SIGWINCH/`TIOCSWINSZ`): newly-exposed cells blanked, scroll region reset, cursor clamped, no reflow (matches xterm/VT). Backing store is fixed-max so this is a dimension change, not a realloc. +9 grid assertions.
- **Live demo**: `programs/pty_demo.cyr` — runs `/bin/ls /` inside a PTY sized to the grid and re-renders the captured output (also demonstrates winsize propagation: `ls` columnates to the grid width).

## [0.1.0] — 2026-06-18

First cut: the headless, platform-agnostic VT core. Fully unit-tested on the
Linux host; no PTY / rendering yet (those are M2 / M3).

### Added
- Initial project scaffold (`cyrius init puka`); cyrius pin `6.2.21`.
- **VT parser** (`src/parser.cyr`): Paul Williams DEC ANSI parser state machine, pure (bytes in → one typed event out via `vt_feed`, no I/O / no rendering / no hot-path allocation). GROUND/ESCAPE/CSI/OSC fully handled; DCS + SOS/PM/APC consumed-and-discarded (full DCS passthrough is M6). Events PRINT/EXECUTE/CSI/ESC/OSC with `vt_param*`/`vt_prefix`/`vt_intermediate`/`vt_string_*` accessors. UTF-8-mode (bytes ≥0x80 printable; no 8-bit C1). `tests/parser.tcyr` — 70 assertions.
- **Cell grid** (`src/grid.cyr`): the screen model and single source of truth — cells packed 2×i64 (glyph+attrs / fg+bg) behind accessors, cursor, scroll region, tab stops; scroll (region + sub-region), erase, insert/delete-cell, bce blanking. `tests/grid.tcyr` — 43 assertions.
- **Unicode** (`src/unicode.cyr`): incremental UTF-8 decoder (RFC 3629; overlong/surrogate/out-of-range → U+FFFD), UTF-8 encoder, `char_width` wcwidth (UAX#11 East-Asian-Wide + main emoji + common combining ranges; sourced). `tests/unicode.tcyr` — 28 assertions.
- **Terminal** (`src/terminal.cyr`): the driver — pumps bytes through the parser, applies VT semantics to the grid. Printing with deferred autowrap + wide-glyph placement; C0 (BS/HT/LF/VT/FF/CR); CSI cursor moves (CUU/CUD/CUF/CUB/CNL/CPL/CHA/VPA/CUP/HVP), ED/EL, IL/DL/ICH/DCH/ECH, SU/SD, DECSTBM, SGR (attrs + 16/256/truecolor), TBC, IRM, save/restore cursor; DEC private modes DECCKM/DECOM/DECAWM/DECTCEM; ESC IND/RI/NEL/DECSC/DECRC/RIS/HTS. Headless text renderer (grid → UTF-8, wide-spacer aware). `tests/terminal.tcyr` — 49 end-to-end assertions. DA/DSR responses + charset designators + alt-screen deferred (need the PTY writer / M6).
- Demo entry (`src/main.cyr`): drives a canned byte stream through the full pipe parse → grid → render.
- Design docs: `CLAUDE.md`, `docs/development/roadmap.md` (M1–M7 + v1.0 criteria + phase-2 command center), `docs/architecture/overview.md`, ADR-0001 (sovereign reimplementation, no libghostty), ADR-0002 (app-first, engine extracted later), `docs/development/state.md`, README. Root docs the scaffolder skipped: `CONTRIBUTING.md`, `SECURITY.md`, `CODE_OF_CONDUCT.md`.

**192 assertions green across 5 test files.**
