# puka — Architecture Overview

> **Last Updated**: 2026-06-18
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
                                   ┌──────────────┐   glyphs ◄── kashi (bitmap)
                                   │  render/*.cyr│              rekha+sadish (vector, later)
                                   │ (FB / window)│   accel  ◄── mabda / ai-hwaccel (later)
                                   └──────────────┘
                                          │
                                          ▼
                          framebuffer (blit#39 on agnos) / KMS-DRM (Linux dev)
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
| `render/fbdev.cyr` | blit the RGB buffer → Linux `/dev/fb0` (any truecolor bpp); pure pack/blit core + guarded device | **M5 ✅** / post-v1.0 (agnos `blit`#39) |
| `input.cyr` | keyboard → escape-sequence encoding (keysym+mods → bytes; xterm `ctlseqs`; bracketed paste); pure, headless | **M4 ✅** |
| `input/evdev.cyr` | Linux `/dev/input` scancodes → `(sym,mods)` → `input_encode`; pure decode + guarded device | **M5 ✅** / post-v1.0 (agnos HID) |
| `main.cyr` / `programs/puka_session.cyr` | wiring + the live interactive loop (evdev → child → grid → fbdev) | M2+ / **M5 ✅** |

## Platform split

Cyrius targets **Linux + agnos** (+ rv64 / bare-metal), not macOS. The core
(`parser` / `unicode` / `grid` / `terminal`) is platform-agnostic and headless.
Only the **edges** are platform-specific:

- **PTY**: Linux pty pair (dev) vs. the AGNOS kernel pty syscall surface (a gap to grow, post-v1.0).
- **Render**: Linux KMS/DRM or a headless text dump (dev) vs. AGNOS `blit`#39 framebuffer (native, the proof-app path — no compositor needed).
- **Input**: Linux evdev/stdin (dev) vs. the AGNOS xHCI/HID path (native).

A **macOS backend** would require a Cyrius Darwin/Mach-O backend — cyrius-side
work, explicitly out of scope for this repo (see [CLAUDE.md § Platform reality](../../CLAUDE.md)).

## Own-the-stack dependencies

Per first-party standards, puka depends on AGNOS crates rather than rolling its own:

| Need | Crate | When |
|---|---|---|
| Bitmap console glyphs (CP437 / PSF) | `kashi` | **M3 ✅ (wired: freestanding `font_data.cyr` core)** |
| Scalable / vector glyphs | `rekha` + `sadish` | post-v1.0 |
| GPU acceleration | `mabda` / `ai-hwaccel` | M6 (optional) |
| Errors / structured logging | `sakshi` | as needed |
| Trust / auth (command center, later) | `sigil` | phase 2 |

## Engine extraction (forward-looking)

puka ships **app-first**. The VT engine (`parser` + `grid` + `terminal` +
`unicode`) is designed as a clean internal boundary so it can extract to a
Sanskrit-named substrate library once the phase-2 command center becomes its
second consumer — the established `yo`→`taar`, `iam`→`mihi`, `darshini`→`darshana`
discipline. Not pre-extracted; the second consumer's needs shape the API honestly.
See [ADR-0002](../adr/0002-build-app-first-extract-engine-later.md).
