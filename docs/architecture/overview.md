# puka — Architecture Overview

> **Last Updated**: 2026-06-19 (0.6.2)
>
> System-level module map and data flow. For *why* a decision was made, see
> [`../adr/`](../adr/). For non-obvious code invariants, see the numbered notes
> in this directory. For *what's next*, see [`../development/roadmap.md`](../development/roadmap.md).

## What puka is

A sovereign, Cyrius-native terminal emulator — the program that hosts a shell,
**interprets** the terminal byte protocol (escape sequences), maintains a screen
model, and paints it. Ghostty is the architecture reference; the work is an
independent Cyrius reimplementation (ADR-0001).

Note the inverse relationship with **`darshana`** (the AGNOS TTY/ANSI primitives
library): darshana *emits* escape sequences for client programs to print; puka
*parses and interprets* them. They sit on opposite ends of the same protocol and
share its vocabulary, but flow in opposite directions.

## Data flow

```
   child process (shell)
          │  writes bytes (text + escape sequences)
          ▼
        ┌─────────┐   reads
        │   PTY   │◄──────────── input.cyr  ◄── keyboard / mouse
        └─────────┘   writes (encoded keys)        (HID / xHCI on agnos)
          │  byte stream
          ▼
   ┌──────────────┐   typed actions
   │  parser.cyr  │──────────────────────┐   (print, C0/C1, CSI, ESC, OSC, DCS)
   │ (state mach) │                       ▼
   └──────────────┘              ┌──────────────────┐
          ▲                      │   terminal.cyr   │  applies actions
     unicode.cyr                 │  (VT semantics)  │
   (UTF-8 + width)               └──────────────────┘
                                          │  mutates
                                          ▼
                                   ┌──────────────┐
                                   │   grid.cyr   │  cells + cursor + modes
                                   │ (THE state)  │  + scrollback + scroll region
                                   └──────────────┘
                                          │  pure function of state
                                          ▼
                                   ┌──────────────┐   glyphs ◄── kashi (bitmap atlas)
                                   │ render/fb.cyr │              rekha+sadish (vector, later)
                                   │ (grid → RGB)  │   GPU    ◄── mabda native-AMD (pgpu_*, paused)
                                   └──────────────┘
                                          │  XRGB8888 (pixfmt.cyr)
                                          ▼
                                   ┌──────────────┐
                                   │  win_* seam  │  window.cyr  (→ aethersafha)
                                   └──────────────┘
                                          │
                                          ▼
              Wayland client → compositor  (Linux desktop / Hyprland — v1)
              · blit#39 framebuffer        (AGNOS native — post-v1.0)
```

Two invariants hold this together:

1. **The parser is pure.** It does no I/O, no rendering, and no allocation in the
   hot path. Bytes in, typed actions out. This makes it trivially testable and
   fuzzable, and keeps the untrusted-input boundary in one auditable place.
2. **The grid is the single source of truth.** `terminal.cyr` is the only writer;
   the renderer is a pure read of grid state. Any backend (headless text dump,
   Linux framebuffer, AGNOS framebuffer, future GPU) renders the *same* grid. The
   grid also owns the **per-row damage bitset** (marked at every write chokepoint,
   consumed + cleared by the renderer each frame), so a frame repaints only the
   rows that changed — kept in the grid, not the renderer, because the writer is
   the only thing that knows what changed. Single-threaded ordering (pump →
   mutate+mark → render → clear) makes this lock-free and tear-free.

## Modules (planned)

| Module | Owns | Milestone |
|---|---|---|
| `parser.cyr` | DEC ANSI state machine (Williams model); byte stream → typed actions | M1 |
| `unicode.cyr` | UTF-8 decode, east-asian width (wcwidth), grapheme clustering | M1 / M6 |
| `grid.cyr` | `Cell` / `Row` / `Screen`, cursor, scroll region, tab stops, scrollback ring | M1 |
| `terminal.cyr` | VT semantics — applies parser actions to the grid (CUP/ED/EL/SGR/DECSTBM/modes/charsets) | M1 |
| `pty.cyr` | PTY pair allocation, child spawn (explicit argv), read/write loop. Platform-split | M2 (Linux) / post-v1.0 (agnos) |
| `render/fb.cyr` | grid → RGB pixel buffer: colour resolution, glyph blit (kashi), cursor, per-row damage, PPM dump | **M3 ✅ (renderer core)** |
| `input.cyr` | keyboard → escape-sequence encoding (keysym+mods → bytes; xterm `ctlseqs`; bracketed paste); pure, headless | **M4 ✅** |
| `platform/window.cyr` | the cross-platform `win_*` window-backend seam (open/present/poll/next-key/close) → extracts to `aethersafha` | **M6 ✅** |
| `platform/wayland/*` | sovereign Wayland client: wire codec, connect/registry/xdg-shell, `wl_seat`/`wl_keyboard`, `wl_shm` present | **M6 ✅** |
| `render/pixfmt.cyr` · `input/keymap.cyr` | RGB→XRGB8888 + damage-aware blit · shared evdev-keycode→bytes bridge | **M6 ✅** |
| `platform/gpu/gpu.cyr` | the `pgpu_*` GPU seam over mabda's native AMD backend (context/target/render/readback→`wl_shm`) | **M6 ✅ (plumbing); cell renderer paused** |
| `render/atlas.cyr` | kashi glyphs → 128×256 RGBA8 coverage texture for GPU sampling | **M6 ✅ (bite 8a)** |
| `programs/puka_term.cyr` | desktop daily-driver: `poll(wayland, pty)` loop hosting `$SHELL`; CPU `fb.cyr` render (GPU paused) | **M6 ✅** |
| `render/fbdev.cyr` · `input/evdev.cyr` device · `puka_session.cyr` | Linux framebuffer/evdev *console* edges | M5 — **superseded** by Wayland (retire bite 10) |

## Platform split

Cyrius targets **Linux + agnos** (+ rv64 / bare-metal), not macOS. The core
(`parser` / `unicode` / `grid` / `terminal`) is platform-agnostic and headless.
Only the **edges** are platform-specific:

- **PTY**: Linux pty pair vs. the AGNOS kernel pty syscall surface (post-v1.0).
- **Window / surface** (the `win_*` seam, → `aethersafha`): a sovereign **Wayland** client (Linux desktop, the v1 target — a window *in* the compositor) vs. the AGNOS `blit`#39 framebuffer (native, no compositor, post-v1.0). X11 / macOS backends fill the same seam later. A headless text / PPM dump (`fb.cyr`) stays for engine tests.
- **Input**: Wayland `wl_keyboard` (Linux desktop) vs. the AGNOS xHCI/HID path (native) — both feed the *same* `evdev__keymap` → `input_encode` bridge.
- **GPU render**: `mabda`'s native AMD backend (sovereign — no FFI). The plumbing (GPU render → CPU readback → `wl_shm`) is shipped + verified; the GPU *cell* renderer is **paused pending mabda** (see [`../development/state.md`](../development/state.md) § Dep gaps), so cells render on CPU `fb.cyr` today. Zero-copy `zwp_linux_dmabuf_v1` is deferred (blocked on a mabda export accessor).

A **macOS / X11 / Windows backend** is a future `win_*` implementation (macOS needs a
Cyrius Darwin/Mach-O backend, cyrius-side; see [CLAUDE.md § Platform reality](../../CLAUDE.md)).

## Own-the-stack dependencies

Per first-party standards, puka depends on AGNOS crates rather than rolling its own:

| Need | Crate | When |
|---|---|---|
| Bitmap console glyphs (CP437 / PSF) | `kashi` | **M3 ✅ (wired: freestanding `font_data.cyr` core)** |
| Scalable / vector glyphs | `rekha` + `sadish` | post-v1.0 |
| GPU rendering | `mabda` (native AMD backend; `wgpu` FFI forbidden) | **M6 — plumbing ✅ (bite 7); cell renderer paused pending mabda** |
| Errors / structured logging | `sakshi` | as needed |
| Trust / auth (command center) | `sigil` | v3 |

## Engine extraction (forward-looking)

puka ships **app-first**. The VT engine (`parser` + `grid` + `terminal` +
`unicode`) is designed as a clean internal boundary so it can extract to a
Sanskrit-named substrate library once the phase-2 command center becomes its
second consumer — the established `yo`→`taar`, `iam`→`mihi`, `darshini`→`darshana`
discipline. Not pre-extracted; the second consumer's needs shape the API honestly.
See [ADR-0002](../adr/0002-build-app-first-extract-engine-later.md).
