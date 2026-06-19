# puka — Roadmap

> Milestone plan through v1.0. State lives in [`state.md`](state.md);
> this file is the sequencing — what ships, in what order, against
> what dependency gates. Architecture: [`../architecture/overview.md`](../architecture/overview.md).

## Shape

puka is built **app-first, engine-extracted-later** (ADR-0002). The terminal
emulator ships as a binary; once the phase-2 command center becomes a second
consumer, the VT engine (parser + grid + terminal state) extracts to a
Sanskrit-named substrate library — the `yo`→`taar` / `iam`→`mihi` pattern. Don't
pre-extract.

The pipeline is built from the **inside out**: the platform-agnostic core
(parser → grid → terminal) first, fully headless-testable on Linux (M1), then the
I/O edges — PTY (M2), render-to-RGB (M3), input-encode (M4) — then the **desktop
window backend** that makes puka a real terminal *window* in a Wayland compositor
(M6). The engine never knows what surface it draws to; a swappable `win_*` backend
(Wayland now; X11 / macOS / AGNOS later) owns the window, destined to extract to the
**`aethersafha`** windowing crate (ADR-0002).

> **Direction note (0.6.0):** 0.5.0 took a `/dev/fb0` + evdev **console** approach
> (M5). A *desktop* terminal is a **compositor client**, not a console — on a Linux
> desktop everything is a window. So 0.6.0 pivots to a sovereign **Wayland** backend
> (a window in Hyprland, M6). The framebuffer/console edges (`fbdev` / `evdev` device
> layers, `puka_session`) are superseded and queued for retirement; the
> platform-agnostic core (M1–M4) and the `fb.cyr` RGB renderer are unchanged + reused.

**AGNOS-native bring-up is post-v1.0.** AGNOS does not yet have a console/desktop
environment to host puka, so v1.0 targets a **Linux Wayland desktop terminal** (a
kitty-class daily driver); the AGNOS console (`blit`#39 framebuffer + xHCI/HID + the
kernel PTY surface) follows behind the same seams once the platform is ready. The
multi-pane coding-agent **command center** (panes hosting `thoth`) is **v3**. The
sovereign-Cyrius, zero-external-code identity is unchanged — only the sequencing.

## v1.0 criteria

- [ ] **Conformance** — passes a curated vttest / `ctlseqs` corpus (cursor, ED/EL, SGR, scroll regions, modes, charsets, alt-screen)
- [ ] **Public engine API frozen** — parser + grid + terminal-state surface documented and tested (this is what the command center will consume; freezing it triggers the engine-lib extraction per ADR-0002)
- [ ] Runs as a **Wayland desktop terminal** on Linux — a real, titled, resizable window in the compositor (Hyprland/wlroots) hosting a shell, GPU-rendered via `mabda`, daily-drivable as a **kitty replacement**. (The AGNOS-native framebuffer console is the headline **post-v1.0** goal.)
- [ ] Test coverage adequate for the surface (parser fuzzed against malformed input; ≥ the 100-assertion floor)
- [ ] Benchmarks captured in `docs/benchmarks.md` (parser throughput, render frame cost)
- [ ] Security audit pass (`docs/audit/YYYY-MM-DD-audit.md`) — parser is the untrusted-input boundary
- [ ] CHANGELOG complete from v0.1.0 onward

## Milestones

### M0 — Scaffold (v0.1.0) — ✅ shipped 2026-06-18

- `cyrius init puka` scaffold landed; hello-world builds + runs on Linux host
- Doc-tree + design docs (CLAUDE, roadmap, architecture overview, ADR-0001/0002) per [first-party-documentation.md](https://github.com/MacCracken/agnosticos/blob/main/docs/development/first-party/first-party-documentation.md)

### M1 — VT parser + cell-grid core — ✅ complete 2026-06-18

The platform-agnostic core. No rendering, no PTY — pure data. **Done:** all four
modules landed (`parser` / `grid` / `unicode` / `terminal`) + headless text
renderer + demo entry; **192 assertions green**. Deferred to later milestones
(documented in state.md): DA/DSR responses, charset designators, alt-screen.

- **Parser** (`parser.cyr`) — Paul Williams DEC ANSI state machine: `GROUND` / `ESCAPE` / `ESC_INTERMEDIATE` / `CSI_ENTRY` / `CSI_PARAM` / `CSI_INTERMEDIATE` / `CSI_IGNORE` / `OSC_STRING` / `DCS_*` / `SOS_PM_APC`. Emits typed actions (print, execute-C0, csi-dispatch, esc-dispatch, osc, dcs hooks).
- **Unicode** (`unicode.cyr`) — UTF-8 decode + east-asian width (wcwidth) so glyphs land in the right number of cells. Grapheme clustering may defer to M6.
- **Grid** (`grid.cyr`) — `Cell` (codepoint + fg/bg + attrs), `Row`, `Screen` (primary), cursor, scroll region, tab stops.
- **Terminal state** (`terminal.cyr`) — consumes parser actions, mutates the grid: cursor moves (CUP/CUU/…), erase (ED/EL), SGR attributes, `DECSTBM`, autowrap (DECAWM), insert (IRM).
- **Headless test renderer** — dumps grid → text so the whole core is unit-testable on Linux.
- **Acceptance**: feed known sequences, assert grid state; first slice of the conformance corpus green.

### M2 — PTY + process plumbing, Linux — ✅ complete 2026-06-18 (0.2.0)

Wire a real byte stream into the parser. **Done:** `src/pty.cyr` —
`/dev/ptmx` open + unlock + `TIOCGPTN`, `fork` + controlling-tty child setup
(`setsid`/`TIOCSCTTY`/`dup2`) + `execve` with explicit argv, bounded
non-blocking `pty_pump` into `term_feed`, `pty_write`/`pty_set_winsize`/
`pty_wait`/`pty_close`. Resize: `grid_resize` + `term_resize` (no reflow).
`tests/pty.tcyr` spawns a real `/bin/echo` and asserts grid output (skip-clean);
`programs/pty_demo.cyr` runs `/bin/ls /` inside a sized PTY and re-renders it
(winsize propagation visible). Linux-guarded; the agnos PTY surface is post-v1.0.

> **Version map:** M*n* → 0.*n*.0. The grid heap-alloc refactor that M1's notes
> anticipated turned out unnecessary for resize (fixed-max backing) — it's a
> deferred optimization, tracked in state.md, not a milestone gate.

### M3 — Framebuffer renderer + glyphs (0.3.0) — ✅ shipped 2026-06-18

The first *visible* terminal. **Done:** `src/render/fb.cyr` — a pure read of the
grid that resolves the full xterm colour model to RGB (default + 16 ANSI + 6×6×6
cube + grayscale + truecolor; bold=bright / dim / reverse / hidden), paints cell
backgrounds, blits real CP437 glyphs via **`kashi`** (freestanding `font_data.cyr`
core, VGA 8×16), draws the cursor block (DECTCEM-aware), tracks **per-row damage**
(marked at every grid write chokepoint) so a frame repaints only changed rows, and
serializes to a PPM (P6) image — the headless, pixel-assertable verification seam.
`tests/render.tcyr` (44 assertions) + `programs/fb_demo.cyr` (→ `puka_frame.ppm`).
Multi-agent adversarial review closed (2 findings fixed + regression-tested).

> **Scope note:** 0.3.0 ships the platform-agnostic renderer. The **live on-screen
> display backend** — a thin blit of the pixel buffer (`fb_buf()` / `fb_width()` /
> `fb_height()`) to a real surface — lands on **Linux (fbdev/KMS) in M5**; the AGNOS
> `blit`#39 native path is **post-v1.0**. Not a rewrite either way.

- **Glyphs** via **`kashi`** — ✅ wired (built-in CP437 fonts; 16/256/truecolor cell colours resolved). Wide CJK cells paint background only until a wider font lands.
- **Damage tracking** (per-row dirty bits) — ✅.
- **Acceptance** (renderer core): the grid renders, cursor visible, colours correct, scrolling repaints correctly — ✅ pixel-tested + visually confirmed. On-screen smoothness is exercised once the M5 display backend lands.

### M4 — Input encoding (0.4.0) — ✅ shipped 2026-06-18

**Done:** `src/input.cyr` — a pure keyboard→escape-sequence ENCODER, byte-exact
against xterm `ctlseqs` and round-tripped through puka's own parser.
`input_encode(sym, mods, out)` covers printable/UTF-8, Ctrl→C0, Alt→ESC-prefix,
cursor keys (CSI / SS3 under DECCKM / `CSI 1;<mod>` when modified), Home/End,
editing & page keys, F1–F12, BackTab, and the xterm modifier formula via the
single `input__xtmod` chokepoint (disjoint keysym range so a codepoint can't
collide with a named key). `input_paste(text, len, cap, out)` wraps bracketed
paste (mode 2004), strips both CSI introducers (ESC + 8-bit `0x9B`) from the
untrusted body, and bounds-checks every write. terminal.cyr gained the DECCKM +
2004 getters. `tests/input.tcyr` (67) + skip-clean `tests/input_pty.tcyr` (real
PTY echo) + `programs/input_demo.cyr`. Adversarial review closed.

> **Scope note:** 0.4.0 ships the platform-agnostic encoder + a proven PTY
> round-trip (encoded keys → child → echo → grid). The **live raw-key SOURCE** +
> interactive read-loop land on **Linux (evdev) in M5**; the AGNOS xHCI/HID source
> is **post-v1.0**. The encoder is `(sym, mods) → bytes`; the device that
> *produces* `(sym, mods)` is the platform edge.

- Mouse tracking (SGR) deferred to M6.
- **Acceptance** (encoder): every key/modifier sequence is byte-exact and parses back through puka's parser; encoded keys drive a real child end-to-end — ✅. The live "type at a keyboard" session lands with the M5 key source.

### M5 — Linux live terminal (0.5.0) — ✅ shipped 2026-06-18

The live edges over the M1–M4 core: puka runs as a **real interactive terminal on a
Linux framebuffer/VT**, no host terminal underneath — the pre-AGNOS proof, and the
first build you can actually *use*. Each edge keeps a headless-testable core + a
Linux-guarded, skip-clean device layer (the M2/M3/M4 discipline).

- **Display** (`src/render/fbdev.cyr`) — ✅ blits `fb_buf()` (24-bit RGB) to `/dev/fb0`: pure pixel-pack + stride/offset/clamp blit generic over 16/24/32-bpp truecolor layouts (via the `FBIOGET_VSCREENINFO` bitfields), tested against an in-memory fake fb; Linux-guarded open/ioctl/mmap/present with the device geometry validated before the blit trusts it. The ABI struct offsets were validated read-only against real hardware (2560×1440 XRGB8888).
- **Key source** (`src/input/evdev.cyr`) — ✅ decodes raw `/dev/input/eventN` `input_event` records via a US-QWERTY keymap with independent L/R modifier tracking → `input_encode` → `pty_write`; `EVIOCGRAB`s the device. Pure decode tested on synthetic events; untrusted-device-bounded.
- **Interactive loop** (`programs/puka_session.cyr`) — ✅ single-threaded busy-poll: evdev → child, `pty_pump` → grid, `fb_render` → `fbdev_present`; spawns `/bin/sh`, exits on child death.
- **Tests**: `tests/fbdev.tcyr` (25) + `tests/evdev.tcyr` (49). Adversarial review closed (4 fixes).

> **Superseded (0.6.0):** the framebuffer/evdev **console** approach was the wrong
> layer for a *desktop* terminal — a Linux desktop hosts everything as compositor
> windows. 0.6.0 pivots to the **Wayland desktop backend** (M6). `fbdev.cyr`, the
> evdev device layer, and `puka_session.cyr` are queued for retirement; `fb.cyr`
> (the RGB renderer) and the evdev **keymap** are kept and reused unchanged.

### M6 — Wayland desktop terminal (0.6.0 → 0.6.x) — the v1 path

puka as a real **window in a Wayland compositor** (Hyprland) hosting the user's
`$SHELL` — the first windowed Cyrius program, speaking the Wayland wire protocol from
scratch (no libwayland / toolkit / FFI). The engine (M1–M4) is untouched; this is a
swappable `win_*` window backend (extractable to `aethersafha`) + a `poll()` loop
around the core. Built bite-by-bite, each verified live on Hyprland.

**Shipped:**

- ✅ **Wire + window + first render** (0.6.0) — the Wayland wire protocol from scratch (connect, `wl_registry`, SCM_RIGHTS fd-passing), a titled xdg-shell window (configure/ack lifecycle, memfd `wl_shm` buffer), and the grid rendered into it (`fb.cyr` → XRGB8888 → `wl_shm`).
- ✅ **Interactive MVP** (0.6.0) — the `poll()` loop over the Wayland fd + the PTY master hosting `$SHELL`; `wl_keyboard` → `input_encode` → child (raw evdev keycodes, no +8); damage-aware repaint (only changed rows).
- ✅ **Resize + shell config** (0.6.1) — reflow on `xdg_toplevel.configure` (`win_resize_apply` + `term_resize` + `fb_resize` + `pty_set_winsize`); grid ceilings raised to `GRID_MAX_COLS`=**480** / `GRID_MAX_ROWS`=**144** (4K; 3-word damage bitset; buffers grow-only); `$SHELL` hosted as a **login shell** with full env inheritance (`.zprofile`/`.zshrc`/starship source).
- ✅ **GPU plumbing foundation** (0.6.2) — the `pgpu_*` seam over `mabda`'s native AMD GFX9 backend: **GPU render → CPU readback → `wl_shm` → Hyprland**, verified live. `mabda` wired as a git dep (3.2.11). Shader-agnostic foundation; **cells still render on CPU `fb.cyr`**.
- ✅ **Glyph atlas** (0.6.2) — `atlas.cyr` packs kashi's 256 CP437 glyphs (8×16) into a 128×256 RGBA8 coverage texture for GPU sampling (the data the cell renderer will sample).
- ✅ **Alternate screen** (0.6.2, DEC 1049/1047/47) — M7 conformance pulled forward (`vim`/`less`/`htop`/`tmux` work): a lazily heap-backed buffer swap.

**Remaining (toward a daily-drivable v1):**

- **Scrollback ring** *(next)* — primary-screen history + a scroll-back viewport; heap ring (like the alt buffer), disabled on the alt screen.
- **GPU cell renderer** — **PAUSED pending mabda.** Plumbing + atlas are done; blocked on (1) the native RT `va_map` **64 KiB-align fix** *(filed in `mabda/docs/development/issues/2026-06-19-native-rt-vamap-64kib-align-einval.md`)* so puka's padding workaround drops, and (2) ideally a higher-level shading API — mabda has **no instanced-vertex path** (none roadmapped; 3.2.x closes at 3.2.13) and shaders are **hand-assembled SPIR-V**, so the full-screen grid+atlas pass is a large hand-authored effort otherwise. Architecture: a **single full-screen pass** (compute or fullscreen-FS) reading the grid as a **storage buffer** + the atlas as a **texture** → render target → readback → `wl_shm`. CPU `fb.cyr` stays the permanent fallback. Wire into `puka_term` behind a runtime opt-in.
- **Zero-copy `zwp_linux_dmabuf_v1`** *(deferred)* — blocked on mabda's PRIME/dmabuf-export accessor (Phase D, unscheduled). Until then the GPU path memcpy's into `wl_shm` (no mabda change needed).
- **Retire the framebuffer/console edges** — delete the superseded `fbdev.cyr`, the `evdev` *device* layer, and `puka_session.cyr`. (`fb.cyr` and the evdev **keymap** — already lifted to `src/input/keymap.cyr` in 0.6.0 — stay and are reused.)
- **Cross-plat:** the `win_*` seam takes X11 / macOS / AGNOS backends later; the Wayland code extracts to **`aethersafha`** when that crate becomes a real second consumer.

**Acceptance**: a daily-drivable kitty replacement — open puka on Hyprland, run a shell + `vim`, resize cleanly, correct colour/glyph/cursor. (GPU-rendered cells are gated on mabda; CPU rendering is daily-drivable today.)

### M7 — Conformance + polish (0.7.0 → 0.9.0)

- Full vttest / `ctlseqs` conformance: scrollback ring, origin mode, charset switching (DEC special graphics / `ESC ( B`), mouse tracking (SGR), DA/DSR query responses, bracketed-paste edge cases, grapheme clustering, selection/clipboard (`wl_data_device`). *(The alternate screen — DEC 1049 — already shipped in 0.6.2.)*
- Fuzz both the VT parser AND the **Wayland wire parser** (a second untrusted-input boundary — compositor messages) against malformed input.

### M8 — Hardening + v1.0

- P(-1) security sweep (the VT parser + the Wayland wire parser are the untrusted-input boundaries); capture benchmarks (parser throughput, frame cost); complete docs.
- **Engine extraction** (ADR-0002): when the command center (v3) arrives as the second consumer, carve parser+grid+terminal into a Sanskrit-named lib, and extract the windowing seam to `aethersafha`. puka becomes a thin consumer of both.

## Post-v1.0 — AGNOS-native bring-up (the proof-app, when AGNOS is ready)

Deferred past v1.0 because **AGNOS does not yet have a console/desktop environment
to host puka**. When it does, it becomes a new **`win_*` backend** alongside the
Wayland one — the platform-agnostic core (M1–M4), the `fb.cyr` renderer, and the v1.0
Wayland terminal are unchanged; this is purely new platform-edge code behind the same
seams (`win_present` for display, `win_next_key`/`input_encode` for input, `pty_*` for
the child). On AGNOS there is no compositor — puka owns the framebuffer directly:

- **AGNOS PTY surface** — the kernel-side pty syscalls puka needs (a Cyrius-native gap, analogous to the `net.cyr` / `vani` agnos-backend gaps; grown per the kernel-growth rules, not POSIX `forkpty` emulation).
- **AGNOS display** — present the pixel buffer via `blit`#39 (the framebuffer proof path, no compositor) — the AGNOS-native `win_present` backend.
- **AGNOS key source** — keystrokes from the xHCI/HID input path → `input_encode` — the AGNOS-native `win_next_key` backend.
- **Acceptance**: puka boots a shell on AGNOS iron/QEMU and is interactive — the DOOM/tracker-class proof milestone, launching agnoshi.

## v3 — command center (post-v1.0, separate repo)

The supacode-equivalent: a multi-pane **worktree coding-agent command center**, one
puka surface per pane, each running a **`thoth`** session in its own git worktree.
**Not** in puka's repo or v1.0 scope — it's a downstream consumer of the extracted
engine + the `aethersafha` windowing crate. Name/scaffold TBD when puka v1.0 lands.

## Out of scope (for v1.0)

- **AGNOS-native console** — post-v1.0 (see above): AGNOS lacks a console/desktop environment to host puka today, so v1.0 is a Linux Wayland desktop terminal and the AGNOS backend follows behind the `win_*` seam when the platform is ready.
- **X11 / macOS / Windows window backends** — designed-for behind the `win_*` seam, but v1.0 ships Wayland only. (Cyrius emits Linux + agnos; macOS needs a Cyrius Darwin backend, cyrius-side.)
- **The command center itself** — v3, separate repo, post-v1.0.
- **Sixel / Kitty / iTerm2 image protocols** — post-v1.0 (the `kii` image-to-ANSI path is adjacent but separate).
- **Ligatures / complex text shaping** — needs `rekha` vector fonts; post-v1.0.
- **Tabs / splits / config UI inside puka** — terminal-multiplexer concerns belong to the command center, not the emulator.
