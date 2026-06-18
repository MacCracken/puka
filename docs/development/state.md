# puka — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.1.0** — scaffolded 2026-06-18 via `cyrius init puka`. No releases yet.
Design phase: docs landed, no domain code yet.

## Toolchain

- **Cyrius pin**: `6.2.21` (in `cyrius.cyml [package].cyrius`) — current at scaffold

## Source

- `src/main.cyr` — scaffold hello-world only. Builds + runs on Linux host.

## Tests

- `tests/puka.tcyr` — primary suite (scaffold smoke; passes on `cyrius test`)
- `tests/puka.bcyr` — benchmark stub
- `tests/puka.fcyr` — fuzz stub

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
