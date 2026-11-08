# 0005 — v1.0 ships on setu, with GPU-rendered cells

**Status**: Accepted
**Date**: 2026-11-08

## Context

ADR 0003 made a Wayland window on Hyprland the v1.0 target and left agnos until after it. The project
then moved on without the documents. From 0.6.4 puka became a client of the aethersafha compositor over
setu. By 0.6.10 a live shell ran in a composited window on agnos, through an agnos PTY backend and a line
discipline. The binary built from `src/main.cyr` and shipped by the release workflow now speaks only
setu. Wayland survives in `programs/puka_term.cyr`, and dhancha, which puka's window is built on, refuses
Wayland by design.

The v1.0 criteria also required GPU-rendered cells, with mabda's 64 KiB `va_map` bug cited as the blocker.
That was fixed in mabda 3.4.2, so the criterion is now a matter of doing the work.

## Decision

**v1.0 is the setu binary**, on Linux and agnos, with GPU-rendered cells through mabda on Linux and the
CPU renderer as the fallback. The Wayland backend stays built and tested as a second `win_*` backend, and
does not gate a release.

## Consequences

- **Positive** — v1.0 describes what ships. The agnos path, already built and proven, is in scope.
- **Negative** — features that need the window now depend on setu and aethersafha. Clipboard and window
  titles have no message on the setu wire yet, so those releases wait on another repo. puka's v1.0 on a
  Linux desktop needs aethersafha running, and only `puka_term` runs on Hyprland.
- **Negative** — GPU cells are the largest single item before 1.0, on AMD GFX9 only. On agnos mabda's
  ioctl path fails closed, so agnos renders on the CPU.
- **Neutral** — the Wayland backend carries a maintenance cost with no release depending on it. Its wire
  parser is still an untrusted-input boundary, and stays tested for as long as it is built.

## Alternatives considered

- **Both backends gate v1.0.** Every window feature would need a seam in both layers, for a backend that
  no release ships.
- **Keep Wayland as the gate.** It would have left the shipped binary and the stated target disagreeing.
- **Retire Wayland.** The least code, but puka would stop running as a window on Hyprland, and nothing
  forces the choice yet. It stays open for later.
- **GPU cells after 1.0.** A smaller 1.0. Rejected: the GPU path is a stated goal of the terminal, and
  the blocker that argued for deferring it is gone.
