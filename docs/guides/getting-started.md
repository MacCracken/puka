# Getting started with puka

puka is a Wayland terminal — a window in a compositor (Hyprland/wlroots) hosting your
shell. This guide gets you building, running, and contributing. Live status lives in
[`../development/state.md`](../development/state.md); the plan in
[`../development/roadmap.md`](../development/roadmap.md).

## Build + run

```sh
cyrius deps                                          # resolve deps (kashi, mabda)
cyrius build programs/puka_term.cyr build/puka_term  # the daily-driver terminal
WAYLAND_DISPLAY=wayland-1 ./build/puka_term          # opens a window hosting $SHELL
```

Run it inside a Wayland session — set `WAYLAND_DISPLAY` to your compositor's socket
(`ls $XDG_RUNTIME_DIR/wayland-*`). The headless demo (`cyrius build src/main.cyr
build/puka`) drives a *canned* byte stream through the pipeline and prints the grid —
handy for engine work, but it is **not** the terminal.

> **⚠ Never write to `/dev/fb0`** — on a dev box it is the live Hyprland desktop
> framebuffer. puka draws into a `wl_shm` buffer the compositor reads; it never
> touches `/dev/fb0`.

## Test

```sh
cyrius test          # [build].test + every tests/*.tcyr (461 assertions across 12 files)
```

The Wayland + GPU paths need a live compositor / AMD GPU, so they're verified by
running `programs/puka_term.cyr` (and `programs/gpu_win_probe.cyr`) on Hyprland, not in
the headless suite.

## Layout

The engine is modular; each `src/*.cyr` `include`s nothing, and the consuming program
`include`s them in dependency order (see any `programs/*.cyr` head).

- `src/parser.cyr` · `grid.cyr` · `unicode.cyr` · `terminal.cyr` — the headless VT engine (parser → grid → terminal semantics). Pure, fully unit-testable.
- `src/render/fb.cyr` — grid → RGB pixels (CPU renderer); `pixfmt.cyr` packs to XRGB8888.
- `src/platform/window.cyr` — the `win_*` window seam; `src/platform/wayland/*` — the sovereign Wayland client.
- `src/platform/gpu/gpu.cyr` — the `pgpu_*` GPU seam over mabda; `src/render/atlas.cyr` — the kashi glyph atlas.
- `src/pty.cyr` — PTY + `$SHELL` spawn; `src/input.cyr` + `src/input/keymap.cyr` — keyboard → escape bytes.
- `programs/puka_term.cyr` — the daily-driver. `programs/{pty,fb,input}_demo.cyr`, `gpu_probe.cyr`, `gpu_win_probe.cyr` — verification programs.
- `src/main.cyr` — the headless demo entry. Programs end with a **bare `_entry();`** (never `var r = main();`), exiting via `SYS_EXIT` — see CLAUDE.md § Key Principles.

## Contributing a change

Small bites — terminal work is large-effort (CLAUDE.md). For each change:

1. Edit the relevant module (or add one + `include` it in the consuming program/test).
2. Add conformance test cases (`tests/<area>.tcyr`) — every new escape sequence gets one.
3. `cyrius test`, `cyrius fmt`, and the **full `cyrius lint` sweep across `src/` + `programs/`** — untracked "deferred"/"later bite" comments and >120-char lines fail lint; cross-reference a roadmap/CHANGELOG entry or add `#skip-lint`.
4. Update CHANGELOG + `state.md`. Bump `VERSION` only at a release cut — **the user handles all git (commit/tag); never use `gh`.**

See [`../adr/template.md`](../adr/template.md) when a non-trivial design choice deserves an ADR.
