# puka

> Hawaiian: *hole / opening / doorway* — you open a puka into the machine.

A sovereign, [Cyrius](https://github.com/MacCracken/cyrius)-native **terminal
emulator**. Ghostty-class — VT/escape-sequence parser, cell grid, scrollback,
font raster, PTY plumbing — written entirely in Cyrius with **zero external
code**. The native AGNOS console, and the foundation of a future worktree
coding-agent command center.

## Status

Pre-release, design phase. Scaffold builds; the VT engine is the next milestone.
See [`docs/development/state.md`](docs/development/state.md) for live status and
[`docs/development/roadmap.md`](docs/development/roadmap.md) for the plan through v1.0.

## Sovereign by design

puka reimplements a solved problem from scratch. **Ghostty** (architecture),
**supacode** (the command-center product shape), and the VT specs (ECMA-48,
DEC STD 070, the Williams ANSI parser, xterm `ctlseqs`) are **prior art and
conformance targets** — studied, never embedded. No FFI, no vendored source, no
shell-out. See [ADR-0001](docs/adr/0001-sovereign-reimplementation-no-external-engine.md).

Cyrius emits Linux + agnos (+ rv64 / bare-metal), so puka runs natively on the
**AGNOS framebuffer** and on Linux for dev. macOS would require a Cyrius Darwin
backend (out of scope here).

## Build

```sh
cyrius deps                              # resolve deps
cyrius build src/main.cyr build/puka     # compile
cyrius test                              # run [build].test + tests/*.tcyr
```

## Architecture

The pipeline, inside-out: **PTY → parser (pure state machine) → terminal
semantics → cell grid (the source of truth) → renderer (framebuffer)**, with
input encoding feeding keystrokes back to the PTY. Two invariants: the parser is
pure (bytes in, typed actions out), and the grid is the single source of truth
(the renderer is a pure read of it). Full module map and data flow:
[`docs/architecture/overview.md`](docs/architecture/overview.md).

## License

GPL-3.0-only
