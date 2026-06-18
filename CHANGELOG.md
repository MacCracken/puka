# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

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
