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

### M2 — PTY + process plumbing, Linux (v0.3.0)

Wire a real byte stream into the parser.

- **PTY** (`pty.cyr`, Linux backend) — allocate a pty pair, spawn a child with explicit argv (`exec_vec`, never a shell string), non-blocking read/write loop.
- Run a real shell (`sh` / agnoshi) headless; snapshot the grid after commands.
- **Acceptance**: launch `sh`, run `ls`, grid reflects output. Resize (SIGWINCH / `TIOCSWINSZ`) propagates.

### M3 — Framebuffer renderer + glyphs (v0.4.0) — first visible terminal

- **Render** (`render/fb.cyr`) — grid → framebuffer. Linux KMS/DRM for dev, AGNOS `blit`#39 for native.
- **Glyphs** via **`kashi`** (bitmap console fonts — CP437 + PSF). Color: 16/256/truecolor cell attributes.
- Damage tracking (dirty cells) so a frame only repaints what changed.
- **Acceptance**: the grid renders, cursor visible, colors correct, scrolling smooth.

### M4 — Input encoding (v0.5.0) — interactive

- **Input** (`input.cyr`) — keyboard → escape sequences: cursor/function keys, modifiers (shift/ctrl/alt), bracketed paste, application-cursor-key mode. Mouse tracking (SGR) may defer to M6.
- On AGNOS, source keystrokes from the xHCI/HID input path.
- **Acceptance**: type in puka, the shell echoes and responds; a full interactive session works on Linux.

### M5 — AGNOS-native bring-up (v0.6.0) — the proof-app

- **AGNOS PTY surface** — the kernel-side pty syscalls puka needs (a Cyrius-native gap, analogous to the `net.cyr` / `vani` agnos-backend gaps; grown per the kernel-growth rules, not POSIX `forkpty` emulation).
- Run puka on the AGNOS framebuffer as a real console, launching agnoshi.
- **Acceptance**: puka boots a shell on AGNOS iron/QEMU and is interactive — the DOOM/tracker-class proof milestone for the terminal.

### M6 — Conformance + polish (v0.7.0 → v0.9.0)

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
