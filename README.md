# puka

> Hawaiian: *hole / opening / doorway* — you open a puka into the machine.

A sovereign, [Cyrius](https://github.com/MacCracken/cyrius)-native **terminal
emulator** — VT/escape-sequence parser, cell grid, font raster, PTY plumbing —
written entirely in Cyrius with **zero external code** (no FFI, no vendored
source, no shell-out). v1.0 is a window in the
[aethersafha](https://github.com/MacCracken/aethersafha) compositor, on Linux and
AGNOS, hosting your shell, with GPU-rendered cells through
[`mabda`](https://github.com/MacCracken/mabda). It is the eventual foundation of a
worktree coding-agent command center (v3).

## Status

Pre-1.0. The current version is in [`VERSION`](VERSION), live status in
[`docs/development/state.md`](docs/development/state.md), and the plan through v1.0 in
[`docs/development/roadmap.md`](docs/development/roadmap.md). Cells render on the CPU
today (`fb.cyr`).

## Run

puka is a client of the aethersafha compositor over setu:

```sh
cyrius deps                                  # resolve deps
CYRIUS_DCE=1 cyrius build src/main.cyr build/puka
./build/puka                                 # opens a window hosting your shell
```

`$SETU_SOCKET` overrides where it looks for the compositor. With no compositor, or with
`--demo`, it drives a canned byte stream through the pipeline and prints the grid.

On a Wayland desktop, the second backend is a separate program:

```sh
cyrius build programs/puka_term.cyr build/puka_term
WAYLAND_DISPLAY=wayland-1 ./build/puka_term
```

`cyrius test` runs every `tests/*.tcyr`.

## Architecture

The pipeline, inside-out — the engine never knows what surface it draws to:

```
PTY (pty.cyr, hosts $SHELL)
   → parser.cyr (pure Williams DEC ANSI state machine; bytes → typed actions)
   → terminal.cyr (VT semantics) → grid.cyr (the single source of truth)
   → fb.cyr (pure read of the grid → RGB pixels)
   → win_* seam → setu client (platform/setu/window_setu.cyr) → aethersafha
                → Wayland client (platform/wayland/*) → a Wayland compositor (second backend)
keyboard: compositor key event → keymap.cyr → input.cyr (→ escape bytes) → PTY
GPU (0.9.x, not wired yet): pgpu_* seam (platform/gpu/gpu.cyr) over mabda's native AMD backend
```

Two invariants: the **parser is pure** (bytes in, typed actions out — the
untrusted-input boundary), and the **grid is the single source of truth** (the
renderer is a pure read of it). Full module map + data flow:
[`docs/architecture/overview.md`](docs/architecture/overview.md).

## Sovereign by design

puka reimplements a solved problem from scratch. **Ghostty** (architecture),
**supacode** (the command-center product shape), and the VT specs (ECMA-48, DEC
STD 070, the Williams ANSI parser, xterm `ctlseqs`) are **prior art and
conformance targets** — studied, never embedded. The Wayland protocol is spoken
directly over the socket (no libwayland); the GPU path uses `mabda`'s sovereign
**native AMD** backend only (the `wgpu` FFI backend is forbidden). See
[ADR-0001](docs/adr/0001-sovereign-reimplementation-no-external-engine.md) and
[ADR-0005](docs/adr/0005-v1-ships-on-setu-with-gpu-cells.md) (the v1.0 direction).

## Dependencies

Three git deps, all first-party + sovereign:
[`kashi`](https://github.com/MacCracken/kashi) (bitmap console glyphs),
[`setu`](https://github.com/MacCracken/setu) (the AGNOS display protocol + its
reference client) and [`dhancha`](https://github.com/MacCracken/dhancha) (the widget
toolkit; `sadish` / `rupa` / `rekha` arrive through it). Cyrius stdlib otherwise.
Current pins are in [`state.md`](docs/development/state.md); why each is declared
the way it is, in
[`docs/architecture/001-dependencies.md`](docs/architecture/001-dependencies.md).

`mabda` is not declared yet, because nothing built needs it. The 0.9.x GPU work declares it:
[`docs/architecture/001-dependencies.md`](docs/architecture/001-dependencies.md) has the conditions.

> **⚠ Do not write to `/dev/fb0`** — on a dev box that is the live desktop framebuffer.
> puka is a *compositor client* (it draws into a buffer the compositor reads).

## License

GPL-3.0-only
