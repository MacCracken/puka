# puka — Architecture Overview

> **Last Updated**: 2026-11-08 (0.7.3)
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
                                   │ (grid → RGB)  │   GPU    ◄── mabda native-AMD (pgpu_*, 0.9.x)
                                   └──────────────┘
                                          │  XRGB8888 (pixfmt.cyr)
                                          ▼
                                   ┌──────────────┐
                                   │  win_* seam  │  window.cyr  (→ aethersafha)
                                   └──────────────┘
                                          │
                                          ▼
              setu client → aethersafha        (Linux and agnos — v1.0)
              · Wayland client → compositor    (Linux desktop — second backend)
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

## Modules

| Module | Owns |
|---|---|
| `parser.cyr` | DEC ANSI state machine (Williams model); byte stream → typed actions |
| `unicode.cyr` | UTF-8 decode and encode, east-asian width (wcwidth) |
| `grid.cyr` | cells, cursor, scroll region, tab stops, scrollback ring, alt screen, per-row damage |
| `terminal.cyr` | VT semantics: applies parser actions to the grid (CUP/ED/EL/SGR/DECSTBM/modes) |
| `pty.cyr` | PTY allocation, child spawn (explicit argv), read/write. Linux and agnos arms |
| `line_discipline.cyr` | echo and line editing on agnos, which has no kernel tty layer |
| `render/fb.cyr` | grid → RGB: colour resolution, glyph blit (kashi), cursor, damage, PPM dump |
| `render/pixfmt.cyr` | RGB → XRGB8888 and the damage-aware row blit |
| `render/atlas.cyr` | kashi glyphs → RGBA8 coverage texture for the GPU pass (0.9.x) |
| `input.cyr` | keyboard → escape-sequence encoding and bracketed paste; pure, headless |
| `input/keymap.cyr` | shared evdev-keycode → bytes bridge |
| `ui.cyr` | the dhancha widget tree the window is built from |
| `platform/setu/window_setu.cyr` | the setu `win_*` backend: the window the release binary uses |
| `platform/window.cyr`, `platform/wayland/*` | the Wayland `win_*` backend: wire codec, xdg-shell, `wl_shm` |
| `platform/gpu/gpu.cyr` | the `pgpu_*` seam over mabda's native AMD backend; in no entry point yet |
| `main.cyr` | the release binary: the terminal over setu, falling back to the demo |
| `programs/puka_term.cyr` | the Wayland terminal loop |
| `render/fbdev.cyr`, `input/evdev.cyr` device layer, `programs/puka_session.cyr` | superseded console edges, retired at 0.7.4 |

## Platform split

Cyrius targets **Linux + agnos** (+ rv64 / bare-metal), not macOS. The core
(`parser` / `unicode` / `grid` / `terminal`) is platform-agnostic and headless.
Only the **edges** are platform-specific:

- **PTY**: a Linux pty pair, or an agnos channel in PTY mode with `line_discipline.cyr` on top.
- **Window / surface** (the `win_*` seam): setu to aethersafha on Linux and agnos, the v1.0 target ([ADR-0005](../adr/0005-v1-ships-on-setu-with-gpu-cells.md)), or a sovereign Wayland client as a second backend on Linux.
- **Input**: key events from the compositor, setu or `wl_keyboard`, both feeding the same `evdev__keymap` → `input_encode` bridge.
- **GPU render** (0.9.x): `mabda`'s native AMD backend (sovereign, no FFI), on Linux. On agnos its ioctl path fails closed, so agnos renders on the CPU.

A **macOS / X11 / Windows backend** is a future `win_*` implementation (macOS needs a
Cyrius Darwin/Mach-O backend, cyrius-side; see [CLAUDE.md § Platform reality](../../CLAUDE.md)).

## Own-the-stack dependencies

Per first-party standards, puka depends on AGNOS crates rather than rolling its own:

| Need | Crate | When |
|---|---|---|
| Bitmap console glyphs (CP437) | `kashi` | wired |
| Scalable / vector glyphs | `rekha` + `sadish` | post-v1.0 |
| GPU rendering | `mabda` (native AMD backend; `wgpu` FFI forbidden) | 0.9.x |
| Errors / structured logging | `sakshi` | as needed |
| Trust / auth (command center) | `sigil` | v3 |

## Engine extraction (forward-looking)

puka ships **app-first**. The VT engine (`parser` + `grid` + `terminal` +
`unicode`) is designed as a clean internal boundary so it can extract to a
Sanskrit-named substrate library once the phase-2 command center becomes its
second consumer — the established `yo`→`taar`, `iam`→`mihi`, `darshini`→`darshana`
discipline. Not pre-extracted; the second consumer's needs shape the API honestly.
See [ADR-0002](../adr/0002-build-app-first-extract-engine-later.md).
