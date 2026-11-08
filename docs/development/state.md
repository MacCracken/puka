# puka — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.7.3** (2026-11-08) — see [`../../CHANGELOG.md`](../../CHANGELOG.md). Release narrative belongs
there, not here.

0.7.3 is the P(-1) sweep: a full line-by-line audit, [`../audit/2026-11-08-audit.md`](../audit/2026-11-08-audit.md).
60 distinct findings (6 High, 26 Medium, 28 Low), all but three repaired, and the Wayland wire parser now has headless
tests and a fuzz harness. The roadmap was rewritten as a release plan to 1.0.0, and
[ADR-0005](../adr/0005-v1-ships-on-setu-with-gpu-cells.md) records that v1.0 ships on setu with GPU cells.

0.7.2 before it moved dhancha to 0.11.1, its first tag on puka's cyrius pin, and dropped the three overrides that
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

⛔ **SCOPE, STATED SO IT IS NOT MISREAD.** **Version**, **Toolchain**, **Source**, **Tests**,
**Carry-forward**, **Dependencies**, **Dep gaps** and **Next** were re-derived from the tree on 2026-11-08.
The two agnos notes below (the setu transport and `/bin/puka`) are records of 2026-08-02 and were not re-checked.

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

Line counts at 0.7.3. The data flow is in [`../architecture/overview.md`](../architecture/overview.md).

| file | lines | what |
|---|---:|---|
| `src/parser.cyr` | 364 | VT parser, Williams state machine. Pure: one byte in, one typed event out. |
| `src/unicode.cyr` | 198 | UTF-8 decode and encode (maximal subpart), `char_width`. |
| `src/grid.cyr` | 669 | the screen, the single source of truth: cells, cursor, scroll region, tabs, scrollback ring, alt screen, per-row damage. `GRID_MAX_COLS=480`, `GRID_MAX_ROWS=144`. |
| `src/terminal.cyr` | 741 | VT semantics: parser events to grid mutations, saved cursors, modes, resize. |
| `src/pty.cyr` | 464 | PTY and child spawn, explicit argv. Linux and agnos arms, the shared 64 KiB pump cap, child-exit state. |
| `src/line_discipline.cyr` | 197 | echo and line editing on agnos, which has no kernel tty layer. |
| `src/input.cyr` | 242 | key and paste encoding. Pure. |
| `src/input/keymap.cyr` | 22 | evdev keycode + mods to child bytes, shared by both window backends. |
| `src/input/evdev.cyr` | 258 | the console device layer, retired at 0.7.4, and `evdev__keymap`, which moves to `keymap.cyr` then. |
| `src/render/fb.cyr` | 386 | grid to RGB: colour resolution, kashi glyphs through a CP437 lookup, cursor, damage. |
| `src/render/pixfmt.cyr` | 28 | RGB to XRGB8888 and the damage-aware row blit. |
| `src/render/atlas.cyr` | 64 | kashi glyphs as an RGBA8 coverage texture, for the 0.9.x GPU pass. |
| `src/render/fbdev.cyr` | 221 | Linux framebuffer present, retired at 0.7.4. |
| `src/ui.cyr` | 85 | the dhancha widget tree the window is built from. |
| `src/platform/setu/window_setu.cyr` | 359 | **the release window:** the setu `win_*` backend. |
| `src/platform/window.cyr` | 156 | the Wayland `win_*` backend. |
| `src/platform/wayland/wire.cyr` | 41 | Wayland wire codec. Pure. |
| `src/platform/wayland/client.cyr` | 448 | Wayland connection, registry, xdg-shell, keyboard, the bounded send path. Untrusted-input boundary. |
| `src/platform/wayland/shm.cyr` | 114 | memfd-backed `wl_shm` present buffer. |
| `src/platform/gpu/gpu.cyr` | 171 | the `pgpu_*` seam over mabda. In no entry point until 0.9.0. |
| `src/main.cyr` | 413 | **the release binary:** the terminal over setu, falling back to the demo when no compositor answers. |
| `src/test.cyr` | 11 | the `[build] test` entry. Asserts nothing. |
| `programs/puka_term.cyr` | 165 | the Wayland terminal loop. |
| `programs/puka_setu_term.cyr` | 114 | an earlier setu terminal, consolidated into `main.cyr` at 0.7.4. |
| `programs/puka_setu_probe.cyr` | 71 | setu render probe, consolidated at 0.7.4. |
| `programs/puka_session.cyr` | 93 | the framebuffer session, retired at 0.7.4. |
| `programs/engine_reach.cyr` | 24 | CI's gate on `dist/puka.cyr`. |
| `programs/gpu_probe.cyr` | 163 | GPU render and readback probe. |
| `programs/gpu_win_probe.cyr` | 100 | GPU to Wayland window probe. |
| `programs/wl_probe.cyr` | 38 | Wayland globals probe. |
| `programs/pty_demo.cyr` | 58 | headless PTY demo. |
| `programs/fb_demo.cyr` | 59 | framebuffer demo, writes a PPM. |
| `programs/input_demo.cyr` | 74 | input encoder demo. |
| **total** | **6611** | |

`src/grid.cyr` additionally owns the **per-row damage bitset** (`grid_dirty_rows`),
marked at every write chokepoint and consumed by the renderer
(`grid_row_dirty` / `grid_clear_row_dirty` / `grid_mark_row_dirty` / `grid_mark_all_dirty`).

## Tests

**770 assertions, all green** (`cyrius test`), across 19 `.tcyr` files. Re-counted 2026-11-08. The
runner's closing line reads `20 passed`: it counts files, and since cyrius 6.6.17 it also runs the
`[build] test` entry, `src/test.cyr`, which asserts nothing.

- VT behaviour checked against a real terminal: `vt_semantics.tcyr` (86). Every expected value was compared
  with kitty's parser, and the last group pins where puka deliberately differs.
- Screen edges: `screen_edges.tcyr` (17): glyph lookup, resize, the alt screen, the viewport, memory per frame.
- Wayland: `wayland.tcyr` (44): framing, hostile configures, registry strings, id recycling, stalled and dead peers.

- Engine core: `parser.tcyr` (70), `grid.tcyr` (103 — resize, per-row damage, multi-word bitset rows
  ≥ 64, scrollback ring/viewport, **and the 0.6.22 row/viewport bounds guards**), `unicode.tcyr` (43
  — **incl. the 0.6.22 aborting-byte re-dispatch**), `terminal.tcyr` (77 — DECCKM/2004 getters,
  alt-screen 1049/1047/47, **and the end-to-end UTF-8 re-dispatch**).
- Render: `render.tcyr` (62 — palette/resolve/paint/glyph/cursor/PPM, `fb_resize` grow+shrink,
  **and `fb__fill_rect`'s clamp**), `atlas.tcyr` (14 — kashi glyph atlas, bit-for-bit),
  `fbdev.tcyr` (25), `ui.tcyr` (14), `engine_bundle.tcyr` (11 — drives `dist/puka.cyr` as a consumer
  would, with no PTY, window or app code).
- I/O edge: `input.tcyr` (80 — byte-exact, `vt_feed` round-trips, paste), `evdev.tcyr` (49),
  `line_discipline.tcyr` (49), `pty.tcyr` (10) + `input_pty.tcyr` (2, real PTY echo — both
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
- **`tests/wayland.fcyr`, new at 0.7.3**: 4,000 rounds of messages aimed at the objects that matter, with
  hostile sizes and payloads. **28,000 assertions**, green under `cyrius fuzz --poison`.
- **`tests/puka.bcyr` — five hot-path baselines, real since 0.6.22** (it timed an empty function
  before). Median of five interleaved runs on 2026-11-08 (0.7.3, cyrius 6.7.18): `vt_feed` printable
  10 ns · `vt_feed` CSI SGR 85 ns · `term_feed` full pipeline 202 ns · `grid_scroll_up` 10.30 µs ·
  `fb_render` 80×24 all-dirty 1.23 ms. 0.7.2, alternating with those runs, gives 10 ns, 124 ns,
  195 ns, 10.75 µs and 1.87 ms. ⚠ Baselines, not thresholds — nothing fails on a regression yet.

## Carry-forward / known

- **Large static data warning** (1,147,560 B at 0.7.3): the grid's two `480×144` u64 cell arrays are nearly all of it. Pixel and present buffers are heap or memfd, and grow geometrically since 0.7.3. Acceptable for a desktop binary; heap-allocating the grid remains a deferred optimization.
- **Wide CJK glyphs render blank**: kashi's built-in fonts cover CP437 (0x20..0xFF) only, so a width-2 cell paints its background but no glyph until a wider font (PSF/runtime-loaded or `rekha`) lands. The grid/width handling is already correct.
- **Deferred by the 0.7.3 audit:** Wayland key repeat (D-10, after 1.0), short writes to the PTY master on a large paste (D-12, 0.8.2), and the 32 B per frame under Dep gaps.
- **Not done yet, by release:** see [`roadmap.md`](roadmap.md).

## Dependencies

Declared in `cyrius.cyml` and resolved from their git tags; `cyrius.lock` pins every one by commit.
Why each is declared the way it is, and why mabda is not:
[`../architecture/001-dependencies.md`](../architecture/001-dependencies.md).

| dep | pin (0.7.3) | modules | note |
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
  39 KB → 551 KB). DCE removes most of it. The `--agnos` build is not DCE'd: 1,943,216 B at 0.7.3,
  +14.6% over 0.6.23.

Planned (own-the-stack, not yet wired): `rekha`+`sadish` **directly** (vector glyphs, post-v1.0 — they
are on disk today only as dhancha's transitives), `sakshi` (logging — declared again when puka logs
through it). kashi's PSF/BDF/PCF library face is available but not pulled in (ADR 0004).

## Dep gaps / blockers

- **dhancha allocates in its blit** *(found 2026-11-08, dhancha 0.11.1)* — `dh_canvas_blit_rgb24` calls `alloc(32)`,
  and `alloc` has no free, so each damaged frame leaks 32 B. A one-line fix in dhancha (`var r4[32]`).
  Not inlined here, per CLAUDE.md. `tests/screen_edges.tcyr` holds the line at 32 B.
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
- **mabda had no instanced-vertex path** *(recon 2026-06-19, not re-checked; mabda is not declared until 0.9.0)*, which is why the 0.9.x cell pass is one full-screen pass.
- **mabda had no public dmabuf export** *(recon 2026-06-19, not re-checked)*. Zero-copy GPU present is after 1.0.
- **setu carries no clipboard or window title yet.** They gate 0.8.4 and 0.8.5; see "Needed from other repos" in [`roadmap.md`](roadmap.md).
- **A bare-framebuffer agnos console** is after 1.0. v1.0 on agnos is the aethersafha window.
- **macOS / X11 / Windows:** `win_*` backends designed for and not implemented. v1.0 is setu ([ADR-0005](../adr/0005-v1-ships-on-setu-with-gpu-cells.md)). macOS needs a Cyrius Darwin backend, which is cyrius-side.

## Consumers

_None yet._ The v3 command center (post-v1.0) — `thoth` panes — will be the first; it consumes the extracted engine + `aethersafha`.

## Next

**0.7.4**, retiring the console edges, then the releases in [`roadmap.md`](roadmap.md) through 1.0.0.
