# puka — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.1.0** — scaffolded 2026-06-18 via `cyrius init puka`. No releases yet.
M1 in progress: design docs landed; **Bite 1 (VT parser) complete**.

## Toolchain

- **Cyrius pin**: `6.2.21` (in `cyrius.cyml [package].cyrius`) — current at scaffold

## Source

- `src/parser.cyr` — VT/escape-sequence parser (Williams DEC ANSI state machine). Pure: `vt_feed(byte)` → one typed event. **M1 Bite 1 done.**
- `src/main.cyr` — scaffold hello-world only.

Next modules (M1): `unicode.cyr` (UTF-8 + width), `grid.cyr` (cell/row/screen), `terminal.cyr` (parser events → grid), headless text renderer.

## Tests

- `tests/parser.tcyr` — 70 assertions, all green (`cyrius test`)
- `tests/puka.tcyr` — scaffold smoke (2 assertions)
- `tests/puka.bcyr` / `tests/puka.fcyr` — bench / fuzz stubs (fuzzing the parser is the M1 hardening target)

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

M1 — VT parser + cell-grid core (v0.2.0). See [`roadmap.md`](roadmap.md).
