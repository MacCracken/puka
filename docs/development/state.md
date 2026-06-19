# puka — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.6.0** — **cut 2026-06-18, awaiting user tag.** Contents: **the Wayland desktop
terminal (interactive MVP)** — puka is now a real **window in a Wayland compositor**
(Hyprland) hosting a live `/bin/sh`, the **first windowed Cyrius program**, speaking
the Wayland wire protocol from scratch (no libwayland / toolkit / FFI). New:
`src/platform/window.cyr` (the cross-platform `win_*` backend seam → future
`aethersafha`), `src/platform/wayland/` (`wire.cyr` codec + `client.cyr`
connect/`wl_registry`/xdg-shell/`wl_seat`/`wl_keyboard` + SCM_RIGHTS + `shm.cyr`),
`src/render/pixfmt.cyr` (RGB→XRGB8888 + damage-aware row blit), `src/input/keymap.cyr`
(shared evdev-keycode→bytes bridge), `programs/puka_term.cyr` (a `poll()` loop over
the Wayland fd + the PTY master; `wl_keyboard` → `input_encode` → child; damage-aware
repaint). **Verified live on Hyprland**: typing works, the shell responds, snappy.
This **supersedes** the 0.5.0 framebuffer/console approach (a desktop terminal is a
compositor client, not a `/dev/fb0` console). Gate: **410 headless tests pass**
(engine untouched), fmt/lint clean (0 warnings), VERSION↔cyrius.cyml↔CHANGELOG all
`0.6.0`. The Wayland subsystem is verified live (it needs a compositor → not in the
headless suite). **mabda GPU** rendering + window **resize** are the next bites;
**AGNOS-native** is post-v1.0; the **command center** is v3. (0.5.0 = M5 framebuffer
[superseded]; 0.4.0 = M4 input; 0.3.0 = M3 renderer; 0.2.0 = M2 PTY; 0.1.0 = M1.)
Version map: M(n) → 0.n.0.

## Toolchain

- **Cyrius pin**: `6.2.22` (in `cyrius.cyml [package].cyrius`) — tracks the latest language (kashi 1.0.2's pin).

## Source

- `src/parser.cyr` — VT parser (Williams DEC ANSI state machine). Pure: `vt_feed(byte)` → one typed event.
- `src/grid.cyr` — cell grid (the screen, single source of truth): cells, cursor, scroll region, tabs, scroll/erase/insert/delete primitives.
- `src/unicode.cyr` — UTF-8 decode/encode + `char_width` (wcwidth, UAX#11).
- `src/terminal.cyr` — the driver: parser events → grid mutations (cursor/erase/SGR/scroll/modes/resize) + headless text renderer. Exposes `term_cursor_visible()` (DECTCEM, dirties the cursor row on toggle), `term_app_cursor_get()` (DECCKM) and `term_bracket_paste_get()` (mode 2004) for the renderer/encoder.
- `src/pty.cyr` — PTY + process plumbing (Linux): open/spawn/pump/write/winsize/wait/close. Linux-guarded; the agnos backend is post-v1.0.
- `src/render/fb.cyr` — framebuffer renderer (M3): grid → RGB pixel buffer. Pure read of the grid; colour resolution (default/16/256/truecolor + bold/dim/reverse/hidden), background paint, kashi glyph blit (VGA 8×16), cursor block, per-row damage consumption, PPM (P6) dump. Integer-only, every pixel write bounds-clamped.
- `src/input.cyr` — keyboard→escape-sequence encoder (M4): `input_encode(sym, mods, out)` (disjoint keysym range; xterm modifier formula via `input__xtmod`) + `input_paste(text, len, cap, out)` (bracketed-paste 2004 wrap + ESC/0x9B strip + cap bound). Pure; reads terminal modes via getters.
- **Desktop window backend (M6 / 0.6.0) — the live Wayland edge:**
  - `src/platform/window.cyr` — the cross-platform `win_*` seam (open / present_begin / present_commit / poll_events / next_key / close). Platform-generic names → extracts to `aethersafha`. The engine never references `src/platform/`; mabda's GPU ctx is passed *through* `win_open`.
  - `src/platform/wayland/wire.cyr` — the Wayland wire codec (message framing, u32/string arg encoders). Pure — and the new untrusted-input boundary.
  - `src/platform/wayland/client.cyr` — AF_UNIX connect, `wl_registry` bind, the xdg-shell window lifecycle (configure/ack/ping-pong), `wl_seat`/`wl_keyboard` + a key queue, SCM_RIGHTS fd-passing.
  - `src/platform/wayland/shm.cyr` — memfd-backed `wl_shm` present buffer.
  - `src/render/pixfmt.cyr` — RGB→XRGB8888 pack + damage-aware row blit (the device-neutral core lifted from fbdev).
  - `src/input/keymap.cyr` — shared evdev-keycode→bytes bridge (`wl_keyboard` delivers *raw* evdev keycodes; reuses `evdev__keymap` + the encode discipline).
- `src/render/fbdev.cyr` + `src/input/evdev.cyr` (device layers) — **superseded** by the Wayland backend; queued for retirement (bite 10). The evdev **keymap** is kept + reused; the pure fbdev pack core moved to `pixfmt.cyr`.
- `src/main.cyr` — demo entry: drives a canned stream through the full pipe, prints the rendered grid.
- `programs/puka_term.cyr` — **the desktop daily-driver** (M6): a `poll()` loop over the Wayland fd + the PTY master hosting `/bin/sh`; keyboard → child, child output → grid → damage-aware repaint. Run on Wayland (Hyprland).
- `programs/{pty_demo,fb_demo,input_demo}.cyr` — headless pipe / framebuffer / input demos (engine verification, PPM dumps).
- `programs/puka_session.cyr` — the M5 framebuffer session — **superseded** by `puka_term`; retire bite 10.

`src/grid.cyr` additionally owns the **per-row damage bitset** (`grid_dirty_rows`),
marked at every write chokepoint and consumed by the renderer
(`grid_row_dirty` / `grid_clear_row_dirty` / `grid_mark_row_dirty` / `grid_mark_all_dirty`).

## Tests

- `tests/parser.tcyr` (70), `tests/grid.tcyr` (66, resize + per-row damage), `tests/unicode.tcyr` (28), `tests/terminal.tcyr` (55, + DECCKM/2004 getters), `tests/render.tcyr` (44 — palette/resolve/paint/glyph/cursor/PPM), `tests/input.tcyr` (67 — byte-exact + `vt_feed` round-trips + paste), `tests/fbdev.tcyr` (25 — pixel pack + stride/clamp blit vs a fake fb), `tests/evdev.tcyr` (49 — synthetic-event decode + L/R modifiers), `tests/pty.tcyr` (2) + `tests/input_pty.tcyr` (2, real PTY echo — both skip-clean), `tests/puka.tcyr` (2 smoke) — **410 assertions, all green** (`cyrius test`).
- `tests/puka.bcyr` / `tests/puka.fcyr` — bench / fuzz stubs (fuzzing the parser against adversarial input is the M-hardening target).
- **The Wayland subsystem (M6) has no headless tests** — it needs a live compositor, so it is verified by running `programs/puka_term.cyr` on Hyprland (the pure `wire.cyr` codec gets unit tests against captured byte-vectors in M7). The 410 above are the engine + framebuffer cores.

## Carry-forward / known

- **Large static data warning** (~240KB): grid backing store (~142KB) + kashi's font BSS (~99KB — its glyph tables are u64-unit byte arrays). The renderer's pixel buffer is **heap-allocated** (`alloc`), not static. Acceptable; heap-allocating the grid remains a deferred optimization.
- **Wide CJK glyphs render blank**: kashi's built-in fonts cover CP437 (0x20..0xFF) only, so a width-2 cell paints its background but no glyph until a wider font (PSF/runtime-loaded or `rekha`) lands. The grid/width handling is already correct.
- Deferred (M6 conformance / post-v1.0): DA/DSR query responses, charset designators (ESC ( B), alt-screen (1049), scrollback ring, origin-mode edge cases, mouse tracking, grapheme clustering; non-US keymaps + CapsLock in evdev; AGNOS-native edges (display/input/PTY) are post-v1.0.

## Dependencies

Direct (declared in `cyrius.cyml`):

- stdlib — string, fmt, alloc, io, vec, str, syscalls, assert, bench
- **kashi** — pinned **git dep** (`git`/`tag = "1.0.2"`), **freestanding** `src/font_data.cyr` core only (zero stdlib) — the same core the agnos kernel consumes. Resolves identically on a devbox and in CI (no sibling checkout). Bitmap console glyphs: built-in CP437 VGA 8×16 / CGA 8×8 / VGA 9×16. v1.0.2, API frozen.
- **mabda** — the GPU foundation crate (in `lib/`); puka uses its **native AMD-GFX9 backend only** (sovereign — the `wgpu` FFI backend is forbidden). Wired in M6 bites 7–8 for GPU cell rendering; not yet a build dep (the `wl_shm` CPU MVP ships first).

Planned (own-the-stack, not yet wired): `rekha`+`sadish` (vector glyphs, post-v1.0), `sakshi` (logging). kashi's PSF/BDF/PCF loader library face is available but not pulled in (built-in fonts suffice).

## Dep gaps / blockers

- **mabda dmabuf-export** — mabda has the PRIME `HANDLE_TO_FD` ioctl internally but no public accessor to hand a Wayland client a dmabuf fd (+ fourcc/stride/modifier). Zero-copy GPU present (bite 8 / cut #3) needs a ~30-line `gpu_render_target_export_dmabuf` in mabda. The `wl_shm` software path and the GPU→`wl_shm` memcpy (cut #2) need no mabda change. mabda's native backend is **AMD-GFX9-only** (verified on this Cezanne box) — non-AMD sessions use the `wl_shm` CPU fallback.
- **AGNOS console/desktop environment** — AGNOS can't yet host puka, so **AGNOS-native is post-v1.0** (a new `win_*` backend: `blit`#39 + xHCI/HID + the kernel PTY surface). Not a v1.0 blocker.
- **macOS / X11 / Windows** — `win_*` backends designed-for but not implemented; v1.0 is Wayland-only. macOS needs a Cyrius Darwin backend (cyrius-side).

## Consumers

_None yet._ The v3 command center (post-v1.0) — `thoth` panes — will be the first; it consumes the extracted engine + `aethersafha`.

## Next

**M6 (→ 0.6.x) — finish the Wayland desktop terminal.** The interactive MVP (window +
live shell + correct keyboard + damage-aware render) shipped in 0.6.0. Remaining bites
toward a daily-drivable v1:
1. **Resize** — `xdg_toplevel.configure` → recompute cols/rows → `term_resize` +
   realloc the shm buffer + `pty_set_winsize`; raise `GRID_MAX_*` so a large window
   isn't clamped to 132×64.
2. **mabda GPU** — a kashi glyph atlas + an instanced cell-quad pipeline → a GTT render
   target, presented via `wl_shm` (cut #2), then zero-copy `zwp_linux_dmabuf_v1`
   (cut #3, gated on the mabda export accessor). `wl_shm` stays the permanent CPU fallback.
3. **Retire the framebuffer edges** (bite 10) + a hardening audit of the Wayland wire
   parser (the new untrusted-input boundary).

Then **M7** (conformance: alt-screen, scrollback, charsets, mouse, selection, parser +
wire-parser fuzzing) and **M8** (v1.0 hardening + engine / `aethersafha` extraction).
**AGNOS-native** is post-v1.0; the **command center** is v3. See [`roadmap.md`](roadmap.md).

The grid backing store stays fixed-max (resize is a dim-change), so heap-allocating it
remains a deferred optimization — but **raising `GRID_MAX_*`** is now on the critical
path (a maximized window exceeds 132×64).
