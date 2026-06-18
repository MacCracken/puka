# puka — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.2.0** (in progress) — **0.1.0 released 2026-06-18** (the headless VT core, M1).
0.2.0 cycle open: **M2 (PTY + process plumbing, Linux) COMPLETE** — real
fork/exec through a pseudo-terminal feeds a live child's output into the grid;
resize wired. 203 assertions green. Version map: M(n) → 0.n.0.

## Toolchain

- **Cyrius pin**: `6.2.21` (in `cyrius.cyml [package].cyrius`) — current at scaffold

## Source

- `src/parser.cyr` — VT parser (Williams DEC ANSI state machine). Pure: `vt_feed(byte)` → one typed event.
- `src/grid.cyr` — cell grid (the screen, single source of truth): cells, cursor, scroll region, tabs, scroll/erase/insert/delete primitives.
- `src/unicode.cyr` — UTF-8 decode/encode + `char_width` (wcwidth, UAX#11).
- `src/terminal.cyr` — the driver: parser events → grid mutations (cursor/erase/SGR/scroll/modes/resize) + headless text renderer.
- `src/pty.cyr` — PTY + process plumbing (Linux): open/spawn/pump/write/winsize/wait/close. Linux-guarded; agnos backend is M5.
- `src/main.cyr` — demo entry: drives a canned stream through the full pipe, prints the rendered grid.
- `programs/pty_demo.cyr` — live demo: runs `/bin/ls /` inside a PTY, re-renders the captured grid.

## Tests

- `tests/parser.tcyr` (70), `tests/grid.tcyr` (52, incl. resize), `tests/unicode.tcyr` (28), `tests/terminal.tcyr` (49 end-to-end), `tests/pty.tcyr` (2, real fork/exec through a PTY; skip-clean), `tests/puka.tcyr` (2 smoke) — **203 assertions, all green** (`cyrius test`).
- `tests/puka.bcyr` / `tests/puka.fcyr` — bench / fuzz stubs (fuzzing the parser against adversarial input is the M-hardening target).

## Carry-forward / known

- **Large static data warning** (~142KB): the grid backing store is a fixed module-global array. Acceptable for M1; heap-allocate it when resize lands (M2) — that also enables SIGWINCH.
- Deferred (need the PTY writer or later milestones): DA/DSR query responses, charset designators (ESC ( B), alt-screen (1049), IRM is in but mouse/bracketed-paste modes are M6.

## Dependencies

Direct (declared in `cyrius.cyml`):

- stdlib — string, fmt, alloc, io, vec, str, syscalls, assert, bench

Planned (own-the-stack, not yet wired): `kashi` (bitmap glyphs, M3), `rekha`+`sadish` (vector glyphs, post-v1.0), `mabda`/`ai-hwaccel` (GPU, M6), `sakshi` (logging).

## Dep gaps / blockers

- **AGNOS PTY syscall surface** — kernel-side gap; needed for M5 native bring-up (analogous to the `net.cyr` / `vani` agnos-backend gaps).
- **macOS** — needs a Cyrius Darwin backend; out of scope for this repo.

## Consumers

_None yet._ Phase-2 command center (post-v1.0) will be the first.

## Next

M3 (→ 0.3.0) — framebuffer renderer + `kashi` glyphs: render the grid to a
framebuffer (Linux KMS/DRM for dev, AGNOS `blit`#39 native), the first *visible*
terminal. Then M4 input encoding makes it interactive. See [`roadmap.md`](roadmap.md).

Resize landed without the heap-alloc refactor (the fixed-max backing makes
resize a dim-change); the ~142KB static-data warning therefore remains — heap-
allocating the grid is a deferred optimization, not a blocker.
