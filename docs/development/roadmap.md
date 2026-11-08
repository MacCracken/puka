# puka — Roadmap

> The release plan from here to v1.0. What shipped is in [`CHANGELOG.md`](../../CHANGELOG.md), current
> state is in [`state.md`](state.md), and the architecture is in
> [`../architecture/overview.md`](../architecture/overview.md).

## Shape

puka is built **app-first, engine-extracted-later** (ADR-0002). The terminal ships as a binary. Once the
phase-2 command center becomes a second consumer, the engine (parser + grid + terminal state) extracts to
a Sanskrit-named substrate library. Don't pre-extract.

**v1.0 ships on setu** ([ADR-0005](../adr/0005-v1-ships-on-setu-with-gpu-cells.md)): the binary built from
`src/main.cyr`, a window in the aethersafha compositor on Linux and agnos. The Wayland backend
(`programs/puka_term.cyr`) stays built and tested as a second `win_*` backend, and does not gate a release.

Terminal work is large, so each release below is one small bite, verified before the next. A release
that needs something from another repo is marked ⛔ and moves if that lands late. It does not hold up the
releases after it.

## v1.0 criteria

- [ ] **Conformance** — a curated vttest / `ctlseqs` corpus runs in CI: cursor, ED/EL, SGR, scroll regions, modes, charsets, alt-screen.
- [ ] **A daily-drivable terminal on setu**, on Linux and agnos: a resizable window hosting a shell, correct colour, glyphs and cursor, and full-screen programs working.
- [ ] **GPU-rendered cells** via `mabda` on Linux (AMD GFX9), with `fb.cyr` as the fallback wherever the GPU path is unavailable, which includes agnos today.
- [ ] **Public engine API frozen** — the parser, grid and terminal-state surface is documented and tested.
- [ ] **Tests and fuzzing** — both untrusted-input boundaries (the VT parser and the compositor wire) are fuzzed, and the Wayland backend's tests still pass.
- [ ] **Benchmarks** in `docs/benchmarks.md`: parser throughput and frame cost, CPU and GPU.
- [ ] **A full line-by-line security audit** filed in `docs/audit/`, immediately before 1.0.0.
- [ ] **CHANGELOG** complete from v0.1.0 onward.

## Needed from other repos

| need | owner | status | gates |
|---|---|---|---|
| A clipboard / selection transport | setu + aethersafha | no message exists, not on a roadmap yet | 0.8.4 |
| Window title and app_id | setu + aethersafha | no message exists; aethersafha roadmap D9 | 0.8.5 |
| `sys_chan_winsz_set` for the agnos terminal size | cyrius (open issue), agnos ≥ 1.57.17 | filed, not shipped at cyrius 6.7.18 | 0.8.6 |
| A partial frame then EOF reads as idle forever (audit A-18) | setu | found 2026-11-06, unfixed at 0.9.1 | next setu pin |
| `dh_canvas_blit_rgb24` allocates per frame, leaking 32 B per damaged frame | dhancha | one-line fix, found 2026-11-08 | next dhancha pin |
| `cyrius distlib` hides a bundle's leaves behind a named dep's requires block | cyrius | not filed; worked around with `--no-default-features` | nothing |

## 0.7.x — Hardening, and answering the host

| version | ships |
|---|---|
| **0.7.3** | **P(-1) sweep:** full code audit, security repairs, refactor and optimization. Headless tests and fuzzing for the Wayland wire parser, the last untested boundary. |
| **0.7.4** | **Retire the console edges:** `src/render/fbdev.cyr`, the evdev device layer and `programs/puka_session.cyr`. Consolidate the setu programs that `src/main.cyr` superseded. `fb.cyr` stays, and `evdev__keymap` moves into `input/keymap.cyr`, because `main.cyr` still needs it. |
| **0.7.5** | **A reply path, then DA and DSR:** the engine has no way to answer the host yet. `CSI c`, `CSI > c`, `CSI 5 n` and `CSI 6 n`, so programs that probe the terminal stop waiting on a timeout. |
| **0.7.6** | **Charsets and origin mode:** `ESC (` / `ESC )`, SI/SO, DEC Special Graphics, and the DECOM edge cases. |

The P(-1) sweep that CLAUDE.md requires before each minor runs at the end of each line.

## 0.8.x — Conformance and the pointer

| version | ships |
|---|---|
| **0.8.0** | **The conformance corpus in CI:** vttest-derived fixtures with expected grid state, compared against a real terminal as `tests/vt_semantics.tcyr` was. Every later feature adds its cases here. |
| **0.8.1** | **Mouse reporting:** map setu's pointer motion and buttons (already on the wire), modes 1000/1002/1003 and SGR 1006, and the wheel to applications on the alt screen. |
| **0.8.2** | **Graphemes and paste:** grapheme clustering (combining marks, ZWJ), the bracketed-paste edge cases, and a write queue to the PTY so a large paste cannot be refused part-way. |
| **0.8.3** | **Keyboard:** CapsLock and non-US keymaps. |
| **0.8.4** ⛔ | **Selection and clipboard**, once setu carries it. |
| **0.8.5** ⛔ | **Window titles** (OSC 0 / 2), once setu carries them. |
| **0.8.6** ⛔ | **The agnos terminal size**, so a program in puka sees its own window, once cyrius has the wrapper. |

## 0.9.x — GPU cells and the freeze

The blocker this used to cite, mabda's 64 KiB `va_map` bug, was fixed in mabda 3.4.2. What remains is the
work. The design is a single full-screen pass reading the grid as a storage buffer and the atlas as a
texture, into a render target. `src/render/atlas.cyr` and the `pgpu_*` seam already exist; no entry point
includes the seam yet.

| version | ships |
|---|---|
| **0.9.0** | **Declare mabda:** the restore conditions in `docs/architecture/001-dependencies.md`, including the `color_rgb` collision with `src/terminal.cyr`. The seam builds in an entry point, and agnos still builds, failing closed to the CPU. |
| **0.9.1** | **The GPU cell pass,** proved on a synthetic grid with readback pixel-compared against `fb.cyr`. |
| **0.9.2** | **GPU in the terminal:** into setu's present buffer, with damage, resize, runtime selection and the CPU fallback. |
| **0.9.3** | **Engine API freeze:** the public surface documented and tested, and the docs step of `cyrius audit` passing. |
| **0.9.4** | **Benchmarks and docs:** `docs/benchmarks.md`, the getting-started guide and `docs/examples/`. |

## 1.0.0

A full line-by-line P(-1), not a delta. Then every criterion above is checked against the tree, and the
release is tagged.

## After 1.0

- **Engine extraction** (ADR-0002), when the command center arrives as a second consumer.
- **Wide and vector glyphs** through `rekha` + `sadish`. `kashi` covers CP437 only, so a width-2 cell paints its background and no glyph.
- **Zero-copy GPU present**, once `mabda` has a public export path for it.
- **Wayland depth:** key repeat, pointer, clipboard and dmabuf, to take that backend past a working second window.
- **A bare-framebuffer agnos console**, with no compositor, behind the same `win_*` seam.
- **The v3 command center** (separate repo): a multi-pane worktree agent surface, one puka per pane, each running `thoth`. A downstream consumer of the extracted engine.

## Out of scope for v1.0

- **X11 / macOS / Windows window backends.** Designed for behind the `win_*` seam. macOS needs a Cyrius Darwin backend, which is cyrius-side.
- **Sixel / Kitty / iTerm2 image protocols.**
- **Ligatures and complex text shaping.** They need `rekha` vector fonts.
- **Tabs, splits and a config UI inside puka.** Multiplexing belongs to the command center.
