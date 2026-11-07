# puka — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.7.2** (2026-11-07) — see [`../../CHANGELOG.md`](../../CHANGELOG.md). Release narrative belongs
there, not here.

0.7.2 moves dhancha to 0.11.1, its first tag on puka's cyrius pin, and drops the three overrides that
stood in for it. Behaviour did not move, and `src/` is unchanged.

0.7.1 before it moved to cyrius 6.7.18 and took every other dependency to its latest tag. A new lint
rule found one wrong write length in `programs/`, and CI's lint step now covers `programs/` and `tests/`.

0.7.0 before it moved onto cyrius's 6.7 language line. It was a toolchain release: none of M7's items
has started.

- **Toolchain:** cyrius 6.6.20 → 6.7.9.
- **Dependencies:** each on its tag pinned to 6.7.9, except dhancha, which has no such tag. Its bundle
  was audited here instead.
- **Audit:** the P(-1) delta audit is [`../audit/2026-11-06-audit.md`](../audit/2026-11-06-audit.md),
  with one Low finding, in setu.
- **Engine bundle:** `distlib --check` is back in CI, now that its result does not depend on the
  filesystem. `dist/puka.deps` goes from 4 leaves to 3.
- **GPU seam:** `color_init()` is gone, because mabda 4.2.0 made it a no-op.

0.6.26 before it was the cyrius 6.6.20 and dependency release, with no change under `src/`.

- **Toolchain:** cyrius 6.6.6 → 6.6.20.
- **Dependencies:** each on its tag pinned to 6.6.20, except dhancha, which has no such tag.
- **Engine bundle:** `dist/` now regenerates with `cyrius distlib --no-default-features`. A plain run
  writes a false empty sidecar. `dist/puka.deps` goes from 16 leaves to 4, and CI gates the bundle by
  building `programs/engine_reach.cyr` with `--no-deps`.

0.6.25 before it made three changes and fixed one leak.

- **Compositor loss:** puka exits when the setu compositor goes away (setu's -6 maps to
  `WIN_EV_CLOSE`).
- **Poll leak:** `win_poll_events` no longer leaks 80 B per frame.
- **GPU seam:** it compiles on cyrius 6.6 again.
- **stdlib:** `mmap`, `dynlib` and `sakshi` are no longer declared.

0.6.24 before it was the cyrius 6.6.6 and dependency release. It also cut `cyrius.cyml` to
configuration; the reasoning is in
[`../architecture/001-dependencies.md`](../architecture/001-dependencies.md) and
[ADR 0004](../adr/0004-kashi-freestanding-core-over-library-face.md).

⭐ **0.6.22 IS THIS REPO'S FIRST P(-1) AUDIT** — all 5,788 lines of `src/` read line-by-line, 17
findings filed and repaired: [`../audit/2026-08-31-audit.md`](../audit/2026-08-31-audit.md). Three
were memory-safety defects reachable from data puka does not control (two lazy allocations that
poisoned their own guard and then wrote to the null page; an unvalidated Wayland wire length giving
an out-of-bounds read and an information disclosure). Two P(-1) gates — `cyrius fuzz` and
`cyrius bench` — were passing over **empty harnesses** and now measure something.

⛔ **SCOPE, STATED SO IT IS NOT MISREAD.** **Version**, **Toolchain**, **Dependencies** and **Tests**
were re-derived from the tree on 2026-11-07. The **Source**, **Carry-forward**, **Dep gaps**,
**Consumers** and **Next** sections below still carry the staleness they had at 0.6.7 (they describe
the tree as of 2026-08-02) and were **not** re-verified — the audit read the code, not these
paragraphs. Trust them at that date, not this one.

Since 0.6.7, in brief (the CHANGELOG is authoritative): the **setu/dhancha window + UI edge**
(`win_*` over dhancha's client layer, `src/ui.cyr`), **`dist/puka.cyr` as an embeddable engine**
(0.6.17, `[lib] modules`), the **line discipline** (`src/line_discipline.cyr`), fullscreen/terminal
repairs, the documented **agnos kernel floor 1.56.46** (0.6.20), a **whole-stack dependency refresh**
(0.6.21 — cyrius 6.5.36, dhancha 0.9.26, setu 0.8.8), the **first hardening audit** (0.6.22), and
five toolchain releases (0.6.23 — cyrius 6.6.2; 0.6.24 — cyrius 6.6.6 plus every dependency at its
latest tag; 0.6.26 — cyrius 6.6.20; 0.7.0 — cyrius 6.7.9, the last two with each dependency's tag on
that pin), and 0.7.1 — cyrius 6.7.18 with every dependency at its latest tag.

⛔ **The setu TCP transport is RETIRED (2026-08-03) and puka has NO standing agnos desktop claim on the
replacement.** It is retired as the WRONG PRIMITIVE for a local display protocol — nothing to route,
nothing to checksum, no window to negotiate, no business owning a port — not because it never worked.
Scope the history: **before `net_src_for` (agnos 1.56.34)** it could not complete a compositor↔client
handshake on an ordinary boot, and the only passing test in that era did so because the
`AETHERSAFHA_SETU_SELFTEST` kernel hook assigned `net_ip = 0x7F000001` (hook and script deleted; claims
tracing to them are FALSE GREENS). The replacement transport is the agnos socket (`anu`) — agnos
`docs/development/planning/ipc.md` §9/§10; puka's "first resident" standing must be re-proven there.
⚠ The Linux/Wayland edge is unaffected — a different target, not a fallback.

✅ **What DID happen un-rigged, and is not retracted** (2026-08-02, QEMU `-smp 1`, agnos 1.56.34+):
the honest harness `agnos/scripts/harness/aethersafha-clients-test.py` — which byte-scans `build/agnos`
and hard-exits if the kernel carries any selftest hook — reached **`connected: 2, presented: 2`**, and
**one of the two clients was `/bin/puka`** (setu's slim `present_probe`); the other was the real dhancha
`crab`. ⚠ Scope: QEMU at `-smp 1` only — never shown on iron, and `-smp 4` fault-kills.

⚠ `/bin/puka` as staged for agnos is setu's slim `present_probe`, **not** the full terminal — so the
result above is a *present-path* proof, not a terminal-on-agnos claim, and it rode the now-retired
transport. What is retracted is the pre-`net_src_for` rigged-smoke lineage, not this run.

## Toolchain

- **Cyrius pin**: `6.7.18` (in `cyrius.cyml [package].cyrius`), the same pin every dependency's tag
  carries. Floor: 6.7.7 for
  the `distlib --check` gate in CI; 6.5.8 for the agnos channel band (see the architecture note).
- The `cyrius` wrapper re-executes the pinned toolchain when `~/.cyrius/versions/<pin>` is installed
  (observed 2026-09-27: with 6.6.6 current, a 6.6.2 pin ran 6.6.2's `cycc`).
- ⚠ **`cyrius update` ignores `--help` at 6.6.6 and runs.** It copies the whole stdlib snapshot (111
  files) into `lib/`. Under 6.6.2, `cyrius build --help` and `cyrius test --help` also resolved deps
  before rejecting the flag; 6.6.6 prints their help.
- **`[deps.mabda]` is not declared**, because nothing built needs it. `--agnos` no longer blocks it:
  the agnos peer still has no `SYS_IOCTL`, but since mabda 4.1.5 that call sits behind a wrapper that
  fails closed, and the build links (measured at 6.7.9 with mabda 4.2.0, at 3,246,952 B). Reasoning
  and restore conditions: [`../architecture/001-dependencies.md`](../architecture/001-dependencies.md).
- **`src/platform/gpu/gpu.cyr` is ported to cyrius 6.6's value-form `Result`** (0.6.25). It and both
  GPU probes build again; they failed at 0.6.23 and 0.6.24. ⚠ Build-verified only: neither probe has
  been run on the GPU since the port. They compile against the mabda 4.2.0 the 6.7.18 toolchain
  ships, not the 3.2.11 bite 7 was verified with.
- ⚠ **Kernel floor `agnos >= 1.56.46`** (0.6.20). Below it the terminal opens and every *builtin*
  renders while **every program the shell launches is silent** — a kernel `chan_auth` defect, not a
  puka bug, and indistinguishable from one.

## Source

- `src/parser.cyr` — VT parser (Williams DEC ANSI state machine). Pure: `vt_feed(byte)` → one typed event.
- `src/grid.cyr` — cell grid (the screen, single source of truth): cells, cursor, scroll region, tabs, scroll/erase/insert/delete primitives. Ceilings **`GRID_MAX_COLS=480` / `GRID_MAX_ROWS=144`** (4K window at 8×16); the per-row damage bitset is a **3-word array** (`GRID_DIRTY_WORDS`) so rows past 64 work. `grid_alt_screen` swaps the active cell buffer with a **lazily heap-allocated** backup (the alternate-screen primitive — no static cost until used). **Scrollback**: a lazily heap-allocated ring (`SCROLLBACK_LINES`=1000, primary only) captures top lines on `grid_scroll_up`; `grid_scroll_view`/`grid_view_reset` move a viewport offset; `grid_v*` accessors serve history-or-live cells to the renderer.
- `src/unicode.cyr` — UTF-8 decode/encode + `char_width` (wcwidth, UAX#11).
- `src/terminal.cyr` — the driver: parser events → grid mutations (cursor/erase/SGR/scroll/modes/resize) + headless text renderer. Exposes `term_cursor_visible()` (DECTCEM, dirties the cursor row on toggle), `term_app_cursor_get()` (DECCKM) and `term_bracket_paste_get()` (mode 2004) for the renderer/encoder. **Alternate screen** (DEC 1049/1047/47) via `term__alt_screen` (save cursor → swap → clear → home / restore).
- `src/pty.cyr` — PTY + process plumbing (Linux): open/spawn/pump/write/winsize/wait/close. The child **inherits the full parent environment** (`/proc/self/environ`) with `TERM` overridden to `xterm-256color`; `pty_login_argv0` optionally sets argv[0] (e.g. `-zsh`) for a login shell. Linux-guarded; the agnos backend is post-v1.0.
- `src/render/fb.cyr` — framebuffer renderer (M3): grid → RGB pixel buffer. Pure read of the grid; colour resolution (default/16/256/truecolor + bold/dim/reverse/hidden), background paint, kashi glyph blit (VGA 8×16), cursor block, per-row damage consumption, PPM (P6) dump. Integer-only, every pixel write bounds-clamped. `fb_resize()` refits the buffer to new grid dims (**grow-only** — reused below the high-water mark, the allocator has no `free`). Reads cells through the **viewport-aware** `grid_v*` accessors (history when scrolled back; identical to the live grid at the bottom) and hides the cursor while scrolled back.
- `src/input.cyr` — keyboard→escape-sequence encoder (M4): `input_encode(sym, mods, out)` (disjoint keysym range; xterm modifier formula via `input__xtmod`) + `input_paste(text, len, cap, out)` (bracketed-paste 2004 wrap + ESC/0x9B strip + cap bound). Pure; reads terminal modes via getters.
- **Desktop window backend (M6 / 0.6.0) — the live Wayland edge:**
  - `src/platform/window.cyr` — the cross-platform `win_*` seam (open / present_begin / present_commit / poll_events / next_key / **resize_apply** / close). `win_poll_events` raises `WIN_EV_RESIZE` on a configured size change; `win_resize_apply` adopts it + refits the buffer. Platform-generic names → extracts to `aethersafha`. The engine never references `src/platform/`; mabda's GPU ctx is passed *through* `win_open`.
  - `src/platform/wayland/wire.cyr` — the Wayland wire codec (message framing, u32/string arg encoders). Pure — and the new untrusted-input boundary.
  - `src/platform/wayland/client.cyr` — AF_UNIX connect, `wl_registry` bind, the xdg-shell window lifecycle (configure/ack/ping-pong, `xdg_toplevel.configure` size), `wl_seat`/`wl_keyboard` + a key queue, SCM_RIGHTS fd-passing.
  - `src/platform/wayland/shm.cyr` — memfd-backed `wl_shm` present buffer; `shm_resize` refits it (grow-only memfd/pool, wl_buffer rebuilt at the new dims; old mapping/fd torn down on grow → no leak).
  - `src/render/pixfmt.cyr` — RGB→XRGB8888 pack + damage-aware row blit (the device-neutral core lifted from fbdev).
  - `src/input/keymap.cyr` — shared evdev-keycode→bytes bridge (`wl_keyboard` delivers *raw* evdev keycodes; reuses `evdev__keymap` + the encode discipline).
- **GPU plumbing (M6 bite 7) — the `pgpu_*` seam over mabda's native AMD backend:**
  - `src/render/atlas.cyr` — kashi glyph atlas (bite 8a): packs the 256 CP437 glyphs into a 128×256 RGBA8 coverage texture for GPU sampling. Pure CPU read of kashi; verified bit-for-bit (`tests/atlas.tcyr`, 14). The data the bite-8 shader samples.
  - `src/platform/gpu/gpu.cyr` — puka-generic GPU seam (init / target / render / readback / release). Wraps mabda so the engine/window never bind it directly (the GPU analogue of `win_*` → extracts to `aethersafha`). `pgpu_render` is a placeholder solid fill today; bite 8 swaps in the grid+atlas shader. `pgpu_readback_xrgb` converts the target's RGBA8 → puka's XRGB8888 and is where the cut-#2 memcpy into `wl_shm` lives. Pads the render target to a 64 KiB-aligned size (mabda `va_map` quirk) and reads back the visible sub-rect. **Verified live: GPU → wl_shm → Hyprland** (`programs/gpu_probe.cyr` headless render→readback; `programs/gpu_win_probe.cyr` the full windowed pipe). `puka_term` still renders cells on the CPU — this is the shader-agnostic foundation only.
- `src/render/fbdev.cyr` + `src/input/evdev.cyr` (device layers) — **superseded** by the Wayland backend; queued for retirement (bite 10). The evdev **keymap** is kept + reused; the pure fbdev pack core moved to `pixfmt.cyr`.
- `src/main.cyr` — demo entry: drives a canned stream through the full pipe, prints the rendered grid.
- `programs/puka_term.cyr` — **the desktop daily-driver** (M6): a `poll()` loop over the Wayland fd + the PTY master hosting the user's **`$SHELL` as a login shell** (so `.zprofile`/`.zshrc`/starship source); keyboard → child, child output → grid → damage-aware repaint; **`WIN_EV_RESIZE` → reflow + refit + SIGWINCH + full repaint**. Run on Wayland (Hyprland).
- `programs/{pty_demo,fb_demo,input_demo}.cyr` — headless pipe / framebuffer / input demos (engine verification, PPM dumps).
- `programs/puka_session.cyr` — the M5 framebuffer session — **superseded** by `puka_term`; retire bite 10.

`src/grid.cyr` additionally owns the **per-row damage bitset** (`grid_dirty_rows`),
marked at every write chokepoint and consumed by the renderer
(`grid_row_dirty` / `grid_clear_row_dirty` / `grid_mark_row_dirty` / `grid_mark_all_dirty`).

## Tests

**601 assertions, all green** (`cyrius test`), across 16 `.tcyr` files (0.6.25: 589 + the 12 in
`window_setu.tcyr`). Re-counted 2026-11-07. The runner's closing line reads `17 passed`: it counts
files, and since cyrius 6.6.17 it also runs the `[build] test` entry, `src/test.cyr`, which asserts
nothing. At 0.6.24 this block corrected an earlier claim of 604 across 17: the 604 summed that closing
line into the assertions.

- Engine core: `parser.tcyr` (70), `grid.tcyr` (102 — resize, per-row damage, multi-word bitset rows
  ≥ 64, scrollback ring/viewport, **and the 0.6.22 row/viewport bounds guards**), `unicode.tcyr` (43
  — **incl. the 0.6.22 aborting-byte re-dispatch**), `terminal.tcyr` (77 — DECCKM/2004 getters,
  alt-screen 1049/1047/47, **and the end-to-end UTF-8 re-dispatch**).
- Render: `render.tcyr` (62 — palette/resolve/paint/glyph/cursor/PPM, `fb_resize` grow+shrink,
  **and `fb__fill_rect`'s clamp**), `atlas.tcyr` (14 — kashi glyph atlas, bit-for-bit),
  `fbdev.tcyr` (25), `ui.tcyr` (14), `engine_bundle.tcyr` (11 — drives `dist/puka.cyr` as a consumer
  would, with no PTY, window or app code).
- I/O edge: `input.tcyr` (67 — byte-exact, `vt_feed` round-trips, paste), `evdev.tcyr` (49),
  `line_discipline.tcyr` (49), `pty.tcyr` (2) + `input_pty.tcyr` (2, real PTY echo — both
  skip-clean), `puka.tcyr` (2 smoke).
- Window edge: `window_setu.tcyr` (12, 0.6.25). It drives the setu backend's `win_poll_events` over
  a real AF_UNIX `SOCK_SEQPACKET` pair: idle is no event, polling allocates nothing, `SETU_CLOSE` and
  a dropped connection are both `WIN_EV_CLOSE`. Linux-only, skip-clean. The first test of either
  window backend.
- **`tests/puka.fcyr` — the fuzz harness, real since 0.6.22.** Drives `term_feed` (the whole
  untrusted path) with ~820,000 adversarial bytes: 16 targeted sequences plus 400 rounds of
  fixed-seed pseudo-random and escape-biased noise, asserting grid invariants after each.
  **1,297 assertions**, green under `cyrius fuzz` and `cyrius fuzz --poison`. Since cyrius 6.6.18
  `--poison` covers `alloc()`: a one-byte overwrite added to the harness fails it, and plain
  `cyrius fuzz` does not catch that overwrite (checked 2026-11-06). ⛔ It was a stub that fuzzed
  nothing until 0.6.22. ⚠ `cyrius fuzz --bounds` passes and checks nothing here: `src/` has no typed
  arrays, `*T` subscripts or slices for it to guard.
- **`tests/puka.bcyr` — five hot-path baselines, real since 0.6.22** (it timed an empty function
  before). Median of three interleaved runs on 2026-11-07 (0.7.2, cyrius 6.7.18): `vt_feed` printable
  10 ns · `vt_feed` CSI SGR 124 ns · `term_feed` full pipeline 195 ns · `grid_scroll_up` 10.79 µs ·
  `fb_render` 80×24 all-dirty 1.82 ms. 0.7.1, alternating with those runs, gives 10 ns, 122 ns,
  192 ns, 10.75 µs and 1.87 ms. ⚠ Baselines, not thresholds — nothing fails on a regression yet.
- ⛔ **The Wayland subsystem still has no headless tests**, and that is now the most conspicuous gap:
  the 0.6.22 wire-validation fixes (A-03/A-04) are the repairs that most deserve coverage and are
  verified by reading only. `wire.cyr` is pure and the byte-vector tests are already scoped.

## Carry-forward / known

- **Large static data warning** (~1.2MB): grid backing store (~1.08MB — two `480×144` u64 cell arrays, raised in bite 6 from 132×64) + kashi's font BSS (~99KB — its glyph tables are u64-unit byte arrays). The renderer's pixel buffer + the `wl_shm` present buffer are **heap/memfd-allocated** (grow-only), not static. Acceptable for a desktop binary; heap-allocating the grid remains a deferred optimization (would also let `GRID_MAX_*` grow without BSS cost).
- **Wide CJK glyphs render blank**: kashi's built-in fonts cover CP437 (0x20..0xFF) only, so a width-2 cell paints its background but no glyph until a wider font (PSF/runtime-loaded or `rekha`) lands. The grid/width handling is already correct.
- Deferred (M7 conformance / post-v1.0): DA/DSR query responses, charset designators (ESC ( B), origin-mode edge cases, mouse tracking, grapheme clustering, selection/clipboard; non-US keymaps + CapsLock in evdev; AGNOS-native edges (display/input/PTY) are post-v1.0. **Alt-screen (1049/1047/47) and scrollback are done.**

## Dependencies

Declared in `cyrius.cyml` and resolved from their git tags; `cyrius.lock` pins every one by commit.
Why each is declared the way it is, and why mabda is not:
[`../architecture/001-dependencies.md`](../architecture/001-dependencies.md).

| dep | pin (0.7.2) | modules | note |
|---|---|---|---|
| **kashi** | `1.3.0` | `src/font_data.cyr` | the **freestanding** core only, the same core the agnos kernel and dhancha consume. Built-in CP437 VGA 8×16 / CGA 8×8 / VGA 9×16. [ADR 0004](../adr/0004-kashi-freestanding-core-over-library-face.md). |
| **setu** | `0.9.1` | `dist/setu.cyr` | the AGNOS display protocol + reference client. `present` / `poll_input` go here directly. The same tag dhancha declares. |
| **dhancha** | `0.11.1` | `dist/dhancha.cyr` | the widget toolkit. CONNECT / FD / CLOSE route through it; `src/ui.cyr` builds the canvas widget. |
| **sadish** | `0.12.1` | *(transitive, via dhancha)* | software rasteriser. |
| **rupa** | `0.2.1` | *(transitive, via dhancha)* | theme/design tokens. |
| **rekha** | `1.1.2` | *(transitive, via dhancha)* | vector glyph path. |

- Every tag above is that dep's latest, checked 2026-11-07, and all are pinned to cyrius 6.7.18.
- **The app half is optional.** setu and dhancha, which brings sadish, rupa and rekha, sit under the
  default-on `app` feature, so every build resolves them. `--no-default-features` exists for `cyrius distlib` alone:
  [`../architecture/001-dependencies.md`](../architecture/001-dependencies.md) says why.
- ⚠ **A tag bump leaves the old tag's row in `cyrius.lock`.** Since cyrius 6.6.20 that row stays and
  fails closed. To drop it, delete `cyrius.lock` and run `cyrius deps`, which re-pins every dep.
- stdlib: string, fmt, alloc, io, vec, str, syscalls, assert, bench, args, hashmap, tagged, fnptr,
  result, net, chrono. `mmap`, `dynlib` and `sakshi` were removed at 0.6.25. They had been declared
  for mabda's bundle and nothing in puka referenced them. `dist/puka.deps` lists 4 leaves: `syscalls`,
  `alloc`, `string`, `io` (16 at 0.6.25, when the list was seeded from `[deps] stdlib`). The two GPU
  probes include `lib/sakshi.cyr` and `lib/mmap.cyr` themselves, because mabda needs them.
- **The `path` lines are a dev override.** Since cyrius 6.7.6 a default build always builds the tag.
  To build against an unpushed sibling, uncomment one and set `CYRIUS_LOCAL=<name>`.
- **setu 0.8.11's -6 from `setu_client_poll_input` (the compositor gone) is consumed since 0.6.25.**
  `win_poll_events` maps it to `WIN_EV_CLOSE`, so puka exits instead of polling a dead socket.
- The draw stack dhancha pulls in grew at 0.6.24 (`lib/sadish.cyr` 75 KB → 472 KB, `lib/rekha.cyr`
  39 KB → 551 KB). DCE removes most of it. The `--agnos` build is not DCE'd: 1,934,656 B at 0.7.2,
  +14.0% over 0.6.23.

Planned (own-the-stack, not yet wired): `rekha`+`sadish` **directly** (vector glyphs, post-v1.0 — they
are on disk today only as dhancha's transitives), `sakshi` (logging — declared again when puka logs
through it). kashi's PSF/BDF/PCF library face is available but not pulled in (ADR 0004).

## Dep gaps / blockers

- **The agnos terminal has no window size** *(cyrius issue `2026-11-04-agnos-chan-winsz-ops-peer`,
  open)* — agnos `pty_set_winsize` is `return 0`, so a program in puka sees only the console's grid.
  agnos 1.57.17 has the ops, and cyrius 6.7.18 has no wrapper for them yet. Take them when cyrius adds
  `sys_chan_winsz_set`, and do not hand-roll `syscall(97, 6, …)` meanwhile.
- **setu: a compositor that dies after a partial frame reads as idle forever** *(found 2026-11-06, setu
  0.9.0, unchanged at 0.9.1)* — puka keeps running against the dead socket. Reproduced, Low, and setu's to fix: A-18 in
  [`../audit/2026-11-06-audit.md`](../audit/2026-11-06-audit.md).
- **`cyrius distlib` hides a bundle's stdlib leaves behind a named dep's requires block** *(found
  2026-11-06, cyrius 6.6.20, still so at 6.7.18)* — worked around with `--no-default-features`, and not
  filed in cyrius yet.
- **mabda native RT `va_map` 64 KiB-align bug** *(found 2026-06-19)* — `native_rt_create_2d_rgba8`'s `va_map` returns `EINVAL` unless the BO byte-size is 64 KiB-aligned (256²/512²/1024²/2048² pass; `1260×682×4` = 3360 KiB does not — the GTT BO alloc itself is fine, only the VA map fails). Worked around in `pgpu_target` (pad the alloc to a 256-px multiple per axis, read back the visible sub-rect). The clean fix is mabda-side: round the GTT/`va_map` size up to 64 KiB. Reported to the maintainer.
- **mabda no instanced-vertex path** *(recon 2026-06-19)* — and none is roadmapped (3.2.x closes at 3.2.13; 3.3 = asset-loading). So bite 8 renders the grid via a **single full-screen pass** (texture sampling shipped 3.2.2–3.2.3 + the SPIR-V→GFX9 compiler is complete at 3.2.11, both HW-verified on Cezanne) reading the grid as a storage buffer + the kashi atlas as a texture — NOT instanced quads.
- **mabda dmabuf-export** — mabda has the PRIME `HANDLE_TO_FD` ioctl internally but no public accessor to hand a Wayland client a dmabuf fd (+ fourcc/stride/modifier). Zero-copy GPU present (bite 9 / cut #3) needs a `gpu_render_target_export_dmabuf` in mabda (Phase D, unscheduled). The `wl_shm` GPU→CPU readback memcpy (cut #2, **shipped in bite 7**) needs no mabda change. mabda's native backend is **AMD-GFX9-only** (verified on this Cezanne box) — non-AMD sessions use the `wl_shm` CPU fallback.
- **AGNOS console/desktop environment** — AGNOS can't yet host puka, so **AGNOS-native is post-v1.0** (a new `win_*` backend: `blit`#39 + xHCI/HID + the kernel PTY surface). Not a v1.0 blocker.
- **macOS / X11 / Windows** — `win_*` backends designed-for but not implemented; v1.0 is Wayland-only. macOS needs a Cyrius Darwin backend (cyrius-side).

## Consumers

_None yet._ The v3 command center (post-v1.0) — `thoth` panes — will be the first; it consumes the extracted engine + `aethersafha`.

## Next

**M6 (→ 0.6.x) — finish the Wayland desktop terminal.** Shipped since 0.6.0: **window
resize** (bite 6), the **`$SHELL` login-shell + env** fix, the **GPU plumbing
foundation** (bite 7 — `pgpu_*` seam, render→`wl_shm`→Hyprland verified) + the **glyph
atlas** (bite 8a), and the **alternate screen** (DEC 1049). In flight:
1. **GPU cell renderer** (bite 8) — **PAUSED pending mabda.** Plumbing + atlas are done,
   but the native path needs the **64 KiB-align `va_map` fix** (filed) and ideally a
   higher-level shading API (no instanced path; a full-screen grid+atlas shader is
   otherwise hand-assembled SPIR-V). Resume when those land. CPU `fb.cyr` renders cells.
2. **Conformance (M7 pulled forward)** — alt-screen ✅, scrollback ✅; next: charset
   designators (ESC ( B), mouse tracking (SGR), DA/DSR responses, selection/clipboard
   (`wl_data_device`). The remaining daily-driver gaps for vim/less/tmux.
3. **Retire the framebuffer edges** (bite 10) + a hardening audit of the Wayland wire
   parser (the new untrusted-input boundary).

Then **M8** (v1.0 hardening + engine / `aethersafha` extraction). **AGNOS-native** is
post-v1.0; the **command center** is v3. See [`roadmap.md`](roadmap.md).

Resize is a grid dim-change (fixed-max backing), so heap-allocating the grid remains a
deferred optimization — now also the lever for raising `GRID_MAX_*` further without the
~1 MB BSS cost (a >4K or multi-monitor-span window still clamps to 480×144).
