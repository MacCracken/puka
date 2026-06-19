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
(parser → grid) first, fully headless-testable on Linux, then the I/O edges
(PTY, render, input), then the AGNOS-native bring-up.

## v1.0 criteria

- [ ] **Conformance** — passes a curated vttest / `ctlseqs` corpus (cursor, ED/EL, SGR, scroll regions, modes, charsets, alt-screen)
- [ ] **Public engine API frozen** — parser + grid + terminal-state surface documented and tested (this is what the command center will consume; freezing it triggers the engine-lib extraction per ADR-0002)
- [ ] Runs as a real interactive console on the **AGNOS framebuffer** (boots a shell, accepts keyboard, renders correctly)
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
(winsize propagation visible). Linux-guarded; agnos PTY is M5.

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
> display backend** — pushing the pixel buffer to a real surface (Linux KMS/DRM or
> fbdev for dev, AGNOS `blit`#39 for native) — is folded into **M5**, since AGNOS is
> the real display target and the buffer→screen edge is a thin blit over the
> renderer's `fb_buf()` / `fb_width()` / `fb_height()`, not a rewrite.

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
> round-trip (encoded keys → child → echo → grid). The **live raw-key SOURCE**
> (Linux evdev / AGNOS xHCI-HID) and the interactive read-loop are folded into
> **M5** alongside the display — the encoder is `(sym, mods) → bytes`; the device
> that *produces* `(sym, mods)` is the platform edge.

- Mouse tracking (SGR) deferred to M6.
- **Acceptance** (encoder): every key/modifier sequence is byte-exact and parses back through puka's parser; encoded keys drive a real child end-to-end — ✅. The live "type at a keyboard" session lands with the M5 key source.

### M5 — AGNOS-native bring-up + on-screen display (0.5.0) — the proof-app

- **AGNOS PTY surface** — the kernel-side pty syscalls puka needs (a Cyrius-native gap, analogous to the `net.cyr` / `vani` agnos-backend gaps; grown per the kernel-growth rules, not POSIX `forkpty` emulation).
- **Display backend** (from M3) — push the renderer's pixel buffer to a real surface: the AGNOS `blit`#39 framebuffer (native, the proof path) + a Linux fbdev/KMS adjunct for the dev host. A thin blit over `fb_buf()` / `fb_width()` / `fb_height()`; the renderer itself is done (M3). Pull forward if an on-screen Linux session is wanted before AGNOS bring-up.
- **Key source** (from M4) — feed `input_encode(sym, mods, …)` from a real raw-key device: Linux evdev (`/dev/input`) for the dev host, the AGNOS xHCI/HID path for native. The encoder is done (M4); this is the device-read edge + the interactive loop (`pty_pump` ⇄ render ⇄ key-read).
- Run puka on the AGNOS framebuffer as a real console, launching agnoshi.
- **Acceptance**: puka boots a shell on AGNOS iron/QEMU and is interactive — the DOOM/tracker-class proof milestone for the terminal.

### M6 — Conformance + polish (0.6.0 → 0.9.0)

- Full vttest / `ctlseqs` conformance pass; scrollback ring; **alternate screen**; origin mode; charset switching (DEC special graphics); mouse tracking; bracketed paste edge cases; grapheme clustering.
- Performance: parser throughput, dirty-cell rendering, optional GPU path via **`mabda`**.
- Fuzz the parser hard against malformed/truncated/adversarial input.

### M7 — Hardening + v1.0

- P(-1) security sweep (parser is the untrusted-input boundary); freeze the engine API; capture benchmarks; complete docs.
- **Engine extraction** (ADR-0002): when the command center arrives as the second consumer, carve parser+grid+terminal into a Sanskrit-named lib; puka becomes a thin consumer of it.

## Phase 2 — command center (post-v1.0, separate repo)

The supacode-equivalent: a multi-pane **worktree coding-agent command center**,
one puka surface per pane, each running a `thoth` session in its own git
worktree. **Not** in puka's repo or v1.0 scope — it's a downstream consumer of
the extracted engine. Name/scaffold TBD when puka v1.0 lands.

## Out of scope (for v1.0)

- **macOS / Windows backends** — Cyrius emits Linux + agnos only; macOS needs a Cyrius Darwin backend (cyrius-side, not this repo).
- **The command center itself** — phase 2, separate repo, post-v1.0.
- **Sixel / Kitty / iTerm2 image protocols** — post-v1.0 (the `kii` image-to-ANSI path is adjacent but separate).
- **Ligatures / complex text shaping** — needs `rekha` vector fonts; post-v1.0.
- **Tabs / splits / config UI inside puka** — terminal-multiplexer concerns belong to the command center, not the emulator.
