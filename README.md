# puka

> Hawaiian: *hole / opening / doorway* — you open a puka into the machine.

A sovereign, [Cyrius](https://github.com/MacCracken/cyrius)-native **terminal
emulator** — VT/escape-sequence parser, cell grid, font raster, PTY plumbing —
written entirely in Cyrius with **zero external code** (no FFI, no vendored
source, no shell-out). v1 is a real, resizable **window in a Wayland compositor**
(Hyprland/wlroots) hosting your shell: a daily-drivable **kitty-class terminal**,
GPU-rendered via [`mabda`](https://github.com/MacCracken/mabda). The eventual
foundation of a worktree coding-agent command center (v3) — and, once AGNOS has a
desktop, its native console (post-v1.0).

## Status

**v0.6.2** (see [`VERSION`](VERSION)). The VT engine (M1–M4) is complete; puka runs
as a **Wayland window on Hyprland** hosting `$SHELL` with correct keyboard input,
window resize, the alternate screen (so `vim`/`less`/`tmux` work), and damage-aware
rendering. Cells currently render on the **CPU** (`fb.cyr`); the GPU path is plumbed
and verified end-to-end but the GPU *cell renderer* is paused pending `mabda` (see
the roadmap). **461 headless tests pass.**

The single source of truth for live status is
[`docs/development/state.md`](docs/development/state.md); the plan through v1.0 is
[`docs/development/roadmap.md`](docs/development/roadmap.md).

## Run

puka is a Wayland client — run it inside a Wayland session (Hyprland/wlroots):

```sh
cyrius deps                                          # resolve deps (kashi, mabda)
cyrius build programs/puka_term.cyr build/puka_term  # the daily-driver terminal
WAYLAND_DISPLAY=wayland-1 ./build/puka_term          # opens a window hosting $SHELL
```

`cyrius build src/main.cyr build/puka` builds only the **headless demo** (drives a
canned byte stream through the pipeline and prints the grid) — useful for engine
work, not the terminal. `cyrius test` runs `[build].test` + every `tests/*.tcyr`.

## Architecture

The pipeline, inside-out — the engine never knows what surface it draws to:

```
PTY (pty.cyr, hosts $SHELL)
   → parser.cyr (pure Williams DEC ANSI state machine; bytes → typed actions)
   → terminal.cyr (VT semantics) → grid.cyr (the single source of truth)
   → fb.cyr (pure read of the grid → RGB pixels)
   → win_* seam (window.cyr) → sovereign Wayland client (platform/wayland/*) → compositor
keyboard: wl_keyboard → keymap.cyr → input.cyr (→ escape bytes) → PTY
GPU (optional): pgpu_* seam (platform/gpu/gpu.cyr) over mabda's native AMD backend
```

Two invariants: the **parser is pure** (bytes in, typed actions out — the
untrusted-input boundary), and the **grid is the single source of truth** (the
renderer is a pure read of it). The `win_*` window seam is platform-generic
(Wayland today; X11 / macOS / AGNOS later) and is destined to extract to the
**`aethersafha`** windowing crate. Full module map + data flow:
[`docs/architecture/overview.md`](docs/architecture/overview.md).

## Sovereign by design

puka reimplements a solved problem from scratch. **Ghostty** (architecture),
**supacode** (the command-center product shape), and the VT specs (ECMA-48, DEC
STD 070, the Williams ANSI parser, xterm `ctlseqs`) are **prior art and
conformance targets** — studied, never embedded. The Wayland protocol is spoken
directly over the socket (no libwayland); the GPU path uses `mabda`'s sovereign
**native AMD** backend only (the `wgpu` FFI backend is forbidden). See
[ADR-0001](docs/adr/0001-sovereign-reimplementation-no-external-engine.md) and
[ADR-0003](docs/adr/0003-wayland-desktop-over-framebuffer-console.md) (the v1
direction).

## Dependencies

Two git deps, both first-party + sovereign:
[`kashi`](https://github.com/MacCracken/kashi) (1.0.2 — bitmap console glyphs) and
[`mabda`](https://github.com/MacCracken/mabda) (3.2.11 — GPU foundation, native AMD
GFX9 backend). Cyrius stdlib otherwise. See `state.md` for the dep-gap status.

> **⚠ Do not write to `/dev/fb0`** — on a dev box that is the live Hyprland
> desktop framebuffer. puka is a *compositor client* (it draws into a `wl_shm`
> buffer the compositor reads); it never touches `/dev/fb0`.

## License

GPL-3.0-only
