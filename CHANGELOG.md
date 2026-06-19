# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added
- **GPU plumbing foundation** (M6 bite 7) — puka now drives `mabda`'s native AMD GFX9 backend end-to-end: **GPU render → CPU readback → `wl_shm` → Hyprland window**, verified live (a GPU-rendered frame presented through the compositor, 120 sustained frames, pixel-exact). New `src/platform/gpu/gpu.cyr` — the puka-generic `pgpu_*` seam (init / target / render / readback-with-RGBA8→XRGB8888-swizzle / release), the GPU analogue of the `win_*` seam (extracts to `aethersafha`). `mabda` wired as a git dep at **3.2.11** (the `wgpu` FFI backend stays forbidden; native AMD only); puka's `[deps] stdlib` extended to mabda's superset (`args/hashmap/tagged/fnptr/mmap/dynlib/sakshi`). Probes: `programs/gpu_probe.cyr` (headless render→readback) and `programs/gpu_win_probe.cyr` (the full windowed pipe). **The daily-driver `puka_term` is unchanged — cells still render on the CPU `fb.cyr` path**; this is the shader-agnostic foundation the bite-8 grid renderer builds on.
  - *Architecture correction (recon 2026-06-19):* mabda has **no instanced-vertex path and none is roadmapped** (3.2.x closes at 3.2.13). But texture sampling (3.2.2–3.2.3) and the SPIR-V→GFX9 compiler (3.2.11) are HW-verified on Cezanne, so bite 8 renders the grid via a **single full-screen pass** (compute or fullscreen-FS reading the grid as a storage buffer + the kashi atlas as a texture), not instanced quads.
  - *mabda dep gap found:* the native render target's `va_map` returns `EINVAL` unless the BO byte-size is **64 KiB-aligned** (powers of two pass by luck; `1260×682×4` does not). Worked around in `pgpu_target` by padding the allocated target to a 256-px multiple per axis (always 64 KiB-aligned) and reading back the visible sub-rect. The clean fix is mabda-side (round the GTT/va_map size up to 64 KiB).

## [0.6.1] — 2026-06-19

**Daily-driver polish — resize + real shell config.** The 0.6.0 MVP gains the two
things a kitty replacement needs day one: the window **reflows** on resize (drag /
maximize / tile), and the hosted shell now loads **your** config (`$SHELL` as a login
shell with the full environment, so `.zprofile`/`.zshrc`/starship all source). GPU
rendering gets its own cut next (M6 bites 7–8).

### Added
- **Window resize** (M6 bite 6) — puka now reflows when the compositor resizes the window (drag, maximize, tile). `xdg_toplevel.configure` → `win_poll_events` raises `WIN_EV_RESIZE` → `win_resize_apply` adopts the new size and refits the `wl_shm` present buffer → `term_resize` reflows the grid, `fb_resize` refits the pixel buffer, `pty_set_winsize` SIGWINCHes the child, full repaint. Buffers are **grow-only** (the bump allocator has no `free`, so per-drag realloc would leak; growth converges to the high-water mark — `shm`'s memfd/pool are explicitly torn down on grow, so it never leaks).
- **Raised grid ceilings** — `GRID_MAX_COLS` 132 → **480**, `GRID_MAX_ROWS` 64 → **144** (covers a 4K window at 8×16; even a 1080p window is 67 rows, past the old 64 cap). The per-row damage bitset is now a **3-word array** (was a single u64, which capped rows at 64). Static data grows ~1 MB (the larger cell backing store) — acceptable for a desktop binary.

### Fixed
- **The hosted shell now loads the user's config** (`.zprofile`/`.zshrc`, starship, etc.). Two compounding bugs starved it: `puka_term` execed `/bin/sh` (which never reads `.zshrc`), and `pty_spawn` gave the child an environment of **only** `TERM` — no `$HOME`, `$PATH`, or `$SHELL`, so even zsh couldn't find its rc or resolve `starship`. Now:
  - `pty_spawn` **inherits the full parent environment** (read from `/proc/self/environ`), overriding only `TERM` → `xterm-256color` (puka's advertised capability, not the launching terminal's). Benefits every PTY consumer, not just the desktop loop.
  - `puka_term` execs the user's **`$SHELL`** (fallback `/bin/sh`) as a **login shell** via the new `pty_login_argv0` override (argv[0] = `-zsh`), so the full `.zprofile → .zshrc` chain sources. The auto-typed `/bin/sh` demo banner is retired — the real shell prints its own prompt.
  - Note: powerline/nerd-font glyphs in a starship prompt still render blank — kashi's built-in font is CP437 8×16 only (a separate font-coverage gap, not a config-loading bug).

## [0.6.0] — 2026-06-18

**The Wayland desktop terminal — direction corrected.** puka is now a real window
in a Wayland compositor (Hyprland), hosting a live shell — the **first windowed
program in the Cyrius ecosystem**, speaking the Wayland wire protocol from scratch
(no libwayland / toolkit / FFI). This supersedes the 0.5.0 framebuffer/console
approach: a desktop terminal is a *compositor client*, not a `/dev/fb0` console. v1
is a daily-drivable **kitty replacement** on Linux/Wayland; AGNOS-native framebuffer
is post-v1; the multi-pane coding-agent command center (panes hosting `thoth`) is v3.

### Added
- **`src/platform/window.cyr`** — the cross-platform window-backend seam (`win_open`/`win_present_begin`/`win_present_commit`/`win_poll_events`/`win_next_key`/`win_close`). Platform-generic names (never `wl_*`) so the contract extracts to the future **`aethersafha`** windowing crate as a *move*, not a rewrite; the engine never references `src/platform/`. mabda's GPU ctx is passed *through* `win_open`.
- **`src/platform/wayland/`** — a sovereign Wayland client over the unix socket: `wire.cyr` (the wire codec — 32-bit message framing, string/u32 arg encoders), `client.cyr` (connect, `wl_registry` bind, the full xdg-shell window lifecycle with configure/ack, `wl_seat`/`wl_keyboard`, **SCM_RIGHTS fd-passing**), `shm.cyr` (memfd-backed `wl_shm` present buffer).
- **`programs/puka_term.cyr`** — the desktop daily-driver: a single-threaded `poll()` over the Wayland fd + the PTY master, hosting `/bin/sh`; child output → grid → **damage-aware** repaint (only changed rows), `wl_keyboard` → `input_encode` → child. The interactive CPU-rendered MVP.
- **`src/render/pixfmt.cyr`** — RGB→XRGB8888 packing + a damage-aware row blit (the device-neutral core that survives the fbdev backend's retirement).
- **`src/input/keymap.cyr`** — the shared evdev-keycode→bytes bridge (`wl_keyboard` delivers *raw* evdev keycodes — not evdev+8 — straight into `evdev__keymap` + the encode discipline).
- **kashi → 1.0.2** and **cyrius pin → 6.2.22** (the latest language).

### Notes
- This is the interactive MVP: a window + a live shell + correct keyboard input + snappy damage-aware rendering, verified on Hyprland. Still ahead toward v1: window **resize** (reflow on `xdg_toplevel.configure`), **mabda GPU** rendering (glyph atlas + instanced cell quads, then zero-copy dmabuf), raising the grid ceilings for large windows, and **retiring the 0.5.0 framebuffer edges** (`fbdev`/`evdev` device layers, `puka_session`).
- The Wayland wire parser is a **new untrusted-input boundary** (compositor messages) — a hardening audit is queued alongside the VT parser.

## [0.5.0] — 2026-06-18

**M5 — Linux live terminal.** The live edges over the M1–M4 core: puka now runs as
a real interactive terminal on a Linux framebuffer + evdev keyboard, with no host
terminal underneath — the first build you can actually *use*. The pure cores are
headless-tested; the device layers are Linux-guarded + skip-clean; the on-screen
session runs on a bare Linux VT. (AGNOS-native bring-up is post-v1.0.)

### Added
- **`src/render/fbdev.cyr`** — Linux `/dev/fb0` display backend. Blits the renderer's 24-bit RGB buffer (`fb_buf()`) onto the mmap'd framebuffer, converting to the device's pixel layout (bpp + RGB bitfields from `FBIOGET_VSCREENINFO`) generically over any truecolor format (32/24/16-bpp, stride-aware). The pure pack/blit core is headless-tested against an in-memory fake fb; the open/ioctl/mmap device layer is Linux-guarded. The ABI struct offsets were validated read-only against real hardware (2560×1440 XRGB8888 `amdgpudrmfb`). Device geometry (`bpp ∈ {16,24,32}`, rows-fit-stride, image-fits-mmap) is validated before the blit trusts it.
- **`src/input/evdev.cyr`** — Linux `/dev/input/eventN` raw keyboard source. Decodes 24-byte `input_event` records via a US-QWERTY keymap with independent left/right modifier tracking, feeds `input_encode`, and emits child PTY bytes; `EVIOCGRAB`s the device so keystrokes don't leak to the host VT. The pure decode + keymap core is headless-tested on synthetic events; the open/read device layer is Linux-guarded. Untrusted device input is bounds-checked (whole records only, output headroom, unknown keycodes dropped).
- **`programs/puka_session.cyr`** — the interactive capstone: spawns `/bin/sh` in a PTY, busy-polls the keyboard + master fd (single-threaded, no threads), encodes keys → child, pumps child output → grid, renders dirty rows → framebuffer. Runs on a bare Linux VT (not under X/Wayland).
- **`tests/fbdev.tcyr`** (25) — pixel-pack + stride/offset/clamp blit against a fake framebuffer. **`tests/evdev.tcyr`** (49) — synthetic-event decode: shift/ctrl/alt, L/R-pair release, arrows/F-keys/nav, digit/symbol shift, no-ops.

### Reviewed
- Multi-agent adversarial review (fbdev bounds / untrusted-device / session-loop / idiom), each finding independently verified. Four fixes applied + regression-tested: validate device bpp + geometry before the blit (untrusted screeninfo → OOB guard); independent L/R modifier tracking (a held pair no longer mis-clears on one release); the `evdev_poll` output headroom guard; and the session loop exits on any `waitpid` result (no hang with the keyboard left grabbed).

## [0.4.0] — 2026-06-18

**M4 — input encoding.** The keyboard half of the terminal: a pure, headless
key→escape-sequence encoder, byte-exact against xterm `ctlseqs` and round-tripped
through puka's own parser. 0.4.0 ships the platform-agnostic encoder + a real PTY
round-trip; the live raw-key SOURCE (Linux evdev / AGNOS xHCI-HID) and the
interactive loop ride with the M5 display backend.

### Added
- **`src/input.cyr`** — keyboard→escape-sequence encoder. `input_encode(sym, mods, out)` maps one keystroke to its exact bytes: printable Unicode (UTF-8), Ctrl→C0 fold, Alt/Meta→ESC-prefix; cursor keys (CSI normally, SS3 under DECCKM, `CSI 1;<mod>` when modified); Home/End; editing keys (Insert/Delete/PageUp/PageDown tilde forms); F1–F12 (SS3 P/Q/R/S and the `15/17/18/19/20/21/23/24~` tilde family); BackTab; and the xterm modifier formula `(mods&15)+1` via the single chokepoint `input__xtmod`. Keysyms use a disjoint range (codepoints `0..0x10FFFF`, named keys at `0x110000+`) so a printable scalar can never collide with a named key — the same self-describing-tag idiom as the colour encoding. Reads terminal modes through getters, never globals.
- **`input_paste(text, len, cap, out)`** — bracketed-paste (mode 2004) wrapper: wraps the body in `ESC[200~ … ESC[201~`, strips both CSI introducers (7-bit `ESC` and 8-bit C1 `0x9B`) from the untrusted body so a paste cannot break out of the bracket to inject commands, and bounds-checks every write against `cap`.
- **terminal.cyr**: `term_app_cursor_get()` (DECCKM) and `term_bracket_paste_get()` getters for the encoder; wired DEC private mode 2004 (bracketed paste) in `term__dec_mode` + reset in `term_init`. +6 `terminal.tcyr` assertions.
- **`tests/input.tcyr`** (67) — byte-exact assertions for every key/modifier category + `vt_feed` round-trips (proving emitter and parser agree) + paste wrap/sanitize/cap-guard tests. **`tests/input_pty.tcyr`** (skip-clean) — encoded keystrokes drive a real `/bin/cat` through a PTY and its echo lands in the grid. **`programs/input_demo.cyr`** types two lines into `cat` via the encoder and re-renders.

### Reviewed
- Multi-agent adversarial review (conformance / bounds / untrusted-paste / idiom), each finding independently verified. One low-severity hardening applied: also strip the 8-bit C1 CSI (`0x9B`) from a bracketed paste body (defense-in-depth against a legacy 8-bit-C1 child), with a regression test.

## [0.3.0] — 2026-06-18

**M3 — framebuffer renderer + glyphs.** puka's first *visible* surface. 0.3.0
ships the platform-agnostic renderer (grid → RGB pixel buffer + glyphs + cursor +
per-row damage), verified headlessly via PPM and visually via `fb_demo`. The live
on-screen display backend — pushing this buffer to a real screen — is folded into
the AGNOS-native bring-up (**M5**); on Linux it is a thin fbdev/KMS adjunct over
the same `fb_buf()`.

### Added
- **`src/render/fb.cyr`** — the renderer. A pure read of the grid (single-source-of-truth invariant: the renderer never mutates) that resolves the full xterm colour model to 24-bit RGB — default fg/bg, the 16 ANSI colours, the 16..231 6×6×6 cube (`level(d)=d?d*40+55:0`), the 232..255 grayscale ramp, and truecolor — applying `bold`=bright / `dim` / `reverse` / `hidden` in the correct order (cursor reuses the reverse swap, so SGR-reverse under the cursor cancels). It paints each cell's background rectangle, blits real CP437 glyphs via **kashi**, draws the cursor block (DECTCEM-aware), tracks per-row damage so a frame repaints only changed rows, and serializes the pixel buffer to a PPM (P6) image — the headless, pixel-assertable verification seam (the renderer's analogue of `term_render_row` for text). Integer-only; every pixel write is bounds-clamped (the renderer is a fresh untrusted-input boundary — cell glyph/colour/attr ultimately come from adversarial PTY output). `programs/fb_demo.cyr` renders a styled grid (SGR colours, bold, reverse, 256-colour, truecolor, cursor) to `puka_frame.ppm`.
- **kashi dependency**: wired the **kashi** crate (v1.0.1, API-frozen) as a **pinned git dep** (`git = ".../kashi.git", tag = "1.0.1"`) — selecting only its **freestanding** `src/font_data.cyr` core (zero stdlib: `store8`/`load8` + arithmetic), the same core the agnos kernel consumes. A pinned git dep (rather than a sibling path dep) resolves identically on a devbox and in CI, so no sibling checkout is needed. Gives the built-in CP437 bitmap fonts (VGA 8×16 / CGA 8×8 / VGA 9×16); the PSF/BDF/PCF loader library face is deliberately *not* pulled in (`modules = ["src/font_data.cyr"]`).
- **Per-row damage tracking** (`grid.cyr`): a `u64` dirty-row bitset marked at every grid write chokepoint — cell writes, row copy/blank, insert/delete, cursor moves (mark **both** old and new rows), `init`/`resize` (mark all) — consumed + cleared by the renderer each frame. Public surface: `grid_row_dirty` / `grid_clear_row_dirty` / `grid_mark_row_dirty` / `grid_mark_all_dirty`. +14 `grid.tcyr` assertions.
- **`term_cursor_visible()`** (`terminal.cyr`): DECTCEM visibility accessor for the renderer; the DECTCEM (`?25h`/`?25l`) handler now dirties the cursor row on a real toggle so the block repaints even on an otherwise-quiescent screen.
- **`tests/render.tcyr`** — 44 assertions: palette/cube/grayscale/clamp, token decode, the attribute resolve order, background + cursor pixel paint, the kashi glyph blit ('A' lights pixels, blank cell does not), the DECTCEM re-dirty fix, the `fb_init` geometry-overflow guard, and the PPM P6 header.

### Reviewed
- Multi-agent adversarial review (correctness / bounds & overflow / Cyrius-idiom / untrusted-input), every finding independently verified before acceptance. Fixed two confirmed defects, both regression-tested: a DECTCEM dropped-repaint (a bare cursor hide/show on a settled screen left a ghost / failed to reappear), and a `fb_init` integer-overflow where an out-of-range cell size could wrap the buffer size to a small positive value and undersize the allocation while the plot bounds stayed huge (now cell metrics are capped and the geometry is overflow-checked before `alloc`).

## [0.2.0] — 2026-06-18

### Added
- **M2 — PTY + process plumbing (Linux)**: `src/pty.cyr` — allocate a pseudo-terminal (`/dev/ptmx` → `TIOCSPTLCK` unlock → `TIOCGPTN` → `/dev/pts/N`), `fork` + child-side controlling-tty setup (`setsid` / `TIOCSCTTY` / `dup2` slave→0,1,2) + `execve` with explicit argv (never a shell string), bounded non-blocking pump (`pty_pump`) that reads the master and feeds bytes into `term_feed`, plus `pty_write` (input path, used in M4), `pty_set_winsize`, `pty_wait`, `pty_close`. Linux-guarded (`CYRIUS_TARGET_LINUX`) with non-Linux stubs; agnos PTY is M5. `tests/pty.tcyr` — spawns a real `/bin/echo` and asserts its output lands in the grid (skip-clean if the sandbox blocks `/dev/ptmx`/fork).
- **Resize**: `grid_resize` + `term_resize` — logical screen resize (SIGWINCH/`TIOCSWINSZ`): newly-exposed cells blanked, scroll region reset, cursor clamped, no reflow (matches xterm/VT). Backing store is fixed-max so this is a dimension change, not a realloc. +9 grid assertions.
- **Live demo**: `programs/pty_demo.cyr` — runs `/bin/ls /` inside a PTY sized to the grid and re-renders the captured output (also demonstrates winsize propagation: `ls` columnates to the grid width).

## [0.1.0] — 2026-06-18

First cut: the headless, platform-agnostic VT core. Fully unit-tested on the
Linux host; no PTY / rendering yet (those are M2 / M3).

### Added
- Initial project scaffold (`cyrius init puka`); cyrius pin `6.2.21`.
- **VT parser** (`src/parser.cyr`): Paul Williams DEC ANSI parser state machine, pure (bytes in → one typed event out via `vt_feed`, no I/O / no rendering / no hot-path allocation). GROUND/ESCAPE/CSI/OSC fully handled; DCS + SOS/PM/APC consumed-and-discarded (full DCS passthrough is M6). Events PRINT/EXECUTE/CSI/ESC/OSC with `vt_param*`/`vt_prefix`/`vt_intermediate`/`vt_string_*` accessors. UTF-8-mode (bytes ≥0x80 printable; no 8-bit C1). `tests/parser.tcyr` — 70 assertions.
- **Cell grid** (`src/grid.cyr`): the screen model and single source of truth — cells packed 2×i64 (glyph+attrs / fg+bg) behind accessors, cursor, scroll region, tab stops; scroll (region + sub-region), erase, insert/delete-cell, bce blanking. `tests/grid.tcyr` — 43 assertions.
- **Unicode** (`src/unicode.cyr`): incremental UTF-8 decoder (RFC 3629; overlong/surrogate/out-of-range → U+FFFD), UTF-8 encoder, `char_width` wcwidth (UAX#11 East-Asian-Wide + main emoji + common combining ranges; sourced). `tests/unicode.tcyr` — 28 assertions.
- **Terminal** (`src/terminal.cyr`): the driver — pumps bytes through the parser, applies VT semantics to the grid. Printing with deferred autowrap + wide-glyph placement; C0 (BS/HT/LF/VT/FF/CR); CSI cursor moves (CUU/CUD/CUF/CUB/CNL/CPL/CHA/VPA/CUP/HVP), ED/EL, IL/DL/ICH/DCH/ECH, SU/SD, DECSTBM, SGR (attrs + 16/256/truecolor), TBC, IRM, save/restore cursor; DEC private modes DECCKM/DECOM/DECAWM/DECTCEM; ESC IND/RI/NEL/DECSC/DECRC/RIS/HTS. Headless text renderer (grid → UTF-8, wide-spacer aware). `tests/terminal.tcyr` — 49 end-to-end assertions. DA/DSR responses + charset designators + alt-screen deferred (need the PTY writer / M6).
- Demo entry (`src/main.cyr`): drives a canned byte stream through the full pipe parse → grid → render.
- Design docs: `CLAUDE.md`, `docs/development/roadmap.md` (M1–M7 + v1.0 criteria + phase-2 command center), `docs/architecture/overview.md`, ADR-0001 (sovereign reimplementation, no libghostty), ADR-0002 (app-first, engine extracted later), `docs/development/state.md`, README. Root docs the scaffolder skipped: `CONTRIBUTING.md`, `SECURITY.md`, `CODE_OF_CONDUCT.md`.

**192 assertions green across 5 test files.**
