# puka — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.1.0** — scaffolded 2026-06-18 via `cyrius init puka`. No releases tagged yet.
**M1 (headless VT core) COMPLETE** — parser + grid + unicode + terminal, 192
assertions green, runnable demo. Ready for a first tag (number is the user's call).

## Toolchain

- **Cyrius pin**: `6.2.21` (in `cyrius.cyml [package].cyrius`) — current at scaffold

## Source

- `src/parser.cyr` — VT parser (Williams DEC ANSI state machine). Pure: `vt_feed(byte)` → one typed event.
- `src/grid.cyr` — cell grid (the screen, single source of truth): cells, cursor, scroll region, tabs, scroll/erase/insert/delete primitives.
- `src/unicode.cyr` — UTF-8 decode/encode + `char_width` (wcwidth, UAX#11).
- `src/terminal.cyr` — the driver: parser events → grid mutations (cursor/erase/SGR/scroll/modes) + headless text renderer.
- `src/main.cyr` — demo entry: drives a canned stream through the full pipe, prints the rendered grid.

## Tests

- `tests/parser.tcyr` (70), `tests/grid.tcyr` (43), `tests/unicode.tcyr` (28), `tests/terminal.tcyr` (49 end-to-end), `tests/puka.tcyr` (2 smoke) — **192 assertions, all green** (`cyrius test`).
- `tests/puka.bcyr` / `tests/puka.fcyr` — bench / fuzz stubs (fuzzing the parser against adversarial input is the M1-hardening target).

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

M2 — PTY + process plumbing (Linux): allocate a pty pair, spawn a shell with
explicit argv, wire its output into `term_feed`. First step toward an interactive
terminal. Triggers the grid heap-alloc/resize refactor. See [`roadmap.md`](roadmap.md).
