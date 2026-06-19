# puka — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.6.1** — **cut 2026-06-19, awaiting user tag.** Daily-driver polish on the 0.6.0
Wayland MVP: **window resize** (M6 bite 6 — reflow on drag/maximize/tile via
`WIN_EV_RESIZE` → `win_resize_apply` + `term_resize` + `fb_resize` + `pty_set_winsize`;
grid ceilings raised `GRID_MAX_COLS` 132→**480** / `GRID_MAX_ROWS` 64→**144**, damage
bitset widened to a 3-word array; buffers grow-only) and the **shell-config fix** (the
hosted shell is now the user's **`$SHELL` as a login shell** with the **full inherited
environment** — `TERM` overridden — so `.zprofile`/`.zshrc`/starship source). Gate:
**430 headless tests pass** (was 410; +20 from the multi-word-bitset + `fb_resize`
groups), fmt/lint clean (0 warnings), VERSION↔cyrius.cyml↔CHANGELOG all `0.6.1`. Live
resize (shm refit + present) is verified by dragging on Hyprland (not in the headless
suite). **mabda GPU** rendering is the next cut (bites 7–8); **AGNOS-native** is
post-v1.0; the **command center** is v3. (0.6.0 = Wayland MVP; 0.5.0 = M5 framebuffer
[superseded]; 0.4.0 = M4 input; 0.3.0 = M3 renderer; 0.2.0 = M2 PTY; 0.1.0 = M1.)

## Toolchain

- **Cyrius pin**: `6.2.22` (in `cyrius.cyml [package].cyrius`) — tracks the latest language (kashi 1.0.2's pin).

## Source

- `src/parser.cyr` — VT parser (Williams DEC ANSI state machine). Pure: `vt_feed(byte)` → one typed event.
- `src/grid.cyr` — cell grid (the screen, single source of truth): cells, cursor, scroll region, tabs, scroll/erase/insert/delete primitives. Ceilings **`GRID_MAX_COLS=480` / `GRID_MAX_ROWS=144`** (4K window at 8×16); the per-row damage bitset is a **3-word array** (`GRID_DIRTY_WORDS`) so rows past 64 work.
- `src/unicode.cyr` — UTF-8 decode/encode + `char_width` (wcwidth, UAX#11).
- `src/terminal.cyr` — the driver: parser events → grid mutations (cursor/erase/SGR/scroll/modes/resize) + headless text renderer. Exposes `term_cursor_visible()` (DECTCEM, dirties the cursor row on toggle), `term_app_cursor_get()` (DECCKM) and `term_bracket_paste_get()` (mode 2004) for the renderer/encoder.
- `src/pty.cyr` — PTY + process plumbing (Linux): open/spawn/pump/write/winsize/wait/close. The child **inherits the full parent environment** (`/proc/self/environ`) with `TERM` overridden to `xterm-256color`; `pty_login_argv0` optionally sets argv[0] (e.g. `-zsh`) for a login shell. Linux-guarded; the agnos backend is post-v1.0.
- `src/render/fb.cyr` — framebuffer renderer (M3): grid → RGB pixel buffer. Pure read of the grid; colour resolution (default/16/256/truecolor + bold/dim/reverse/hidden), background paint, kashi glyph blit (VGA 8×16), cursor block, per-row damage consumption, PPM (P6) dump. Integer-only, every pixel write bounds-clamped. `fb_resize()` refits the buffer to new grid dims (**grow-only** — reused below the high-water mark, the allocator has no `free`).
- `src/input.cyr` — keyboard→escape-sequence encoder (M4): `input_encode(sym, mods, out)` (disjoint keysym range; xterm modifier formula via `input__xtmod`) + `input_paste(text, len, cap, out)` (bracketed-paste 2004 wrap + ESC/0x9B strip + cap bound). Pure; reads terminal modes via getters.
- **Desktop window backend (M6 / 0.6.0) — the live Wayland edge:**
  - `src/platform/window.cyr` — the cross-platform `win_*` seam (open / present_begin / present_commit / poll_events / next_key / **resize_apply** / close). `win_poll_events` raises `WIN_EV_RESIZE` on a configured size change; `win_resize_apply` adopts it + refits the buffer. Platform-generic names → extracts to `aethersafha`. The engine never references `src/platform/`; mabda's GPU ctx is passed *through* `win_open`.
  - `src/platform/wayland/wire.cyr` — the Wayland wire codec (message framing, u32/string arg encoders). Pure — and the new untrusted-input boundary.
  - `src/platform/wayland/client.cyr` — AF_UNIX connect, `wl_registry` bind, the xdg-shell window lifecycle (configure/ack/ping-pong, `xdg_toplevel.configure` size), `wl_seat`/`wl_keyboard` + a key queue, SCM_RIGHTS fd-passing.
  - `src/platform/wayland/shm.cyr` — memfd-backed `wl_shm` present buffer; `shm_resize` refits it (grow-only memfd/pool, wl_buffer rebuilt at the new dims; old mapping/fd torn down on grow → no leak).
  - `src/render/pixfmt.cyr` — RGB→XRGB8888 pack + damage-aware row blit (the device-neutral core lifted from fbdev).
  - `src/input/keymap.cyr` — shared evdev-keycode→bytes bridge (`wl_keyboard` delivers *raw* evdev keycodes; reuses `evdev__keymap` + the encode discipline).
- **GPU plumbing (M6 bite 7) — the `pgpu_*` seam over mabda's native AMD backend:**
  - `src/platform/gpu/gpu.cyr` — puka-generic GPU seam (init / target / render / readback / release). Wraps mabda so the engine/window never bind it directly (the GPU analogue of `win_*` → extracts to `aethersafha`). `pgpu_render` is a placeholder solid fill today; bite 8 swaps in the grid+atlas shader. `pgpu_readback_xrgb` converts the target's RGBA8 → puka's XRGB8888 and is where the cut-#2 memcpy into `wl_shm` lives. Pads the render target to a 64 KiB-aligned size (mabda `va_map` quirk) and reads back the visible sub-rect. **Verified live: GPU → wl_shm → Hyprland** (`programs/gpu_probe.cyr` headless render→readback; `programs/gpu_win_probe.cyr` the full windowed pipe). `puka_term` still renders cells on the CPU — this is the shader-agnostic foundation only.
- `src/render/fbdev.cyr` + `src/input/evdev.cyr` (device layers) — **superseded** by the Wayland backend; queued for retirement (bite 10). The evdev **keymap** is kept + reused; the pure fbdev pack core moved to `pixfmt.cyr`.
- `src/main.cyr` — demo entry: drives a canned stream through the full pipe, prints the rendered grid.
- `programs/puka_term.cyr` — **the desktop daily-driver** (M6): a `poll()` loop over the Wayland fd + the PTY master hosting the user's **`$SHELL` as a login shell** (so `.zprofile`/`.zshrc`/starship source); keyboard → child, child output → grid → damage-aware repaint; **`WIN_EV_RESIZE` → reflow + refit + SIGWINCH + full repaint**. Run on Wayland (Hyprland).
- `programs/{pty_demo,fb_demo,input_demo}.cyr` — headless pipe / framebuffer / input demos (engine verification, PPM dumps).
- `programs/puka_session.cyr` — the M5 framebuffer session — **superseded** by `puka_term`; retire bite 10.

`src/grid.cyr` additionally owns the **per-row damage bitset** (`grid_dirty_rows`),
marked at every write chokepoint and consumed by the renderer
(`grid_row_dirty` / `grid_clear_row_dirty` / `grid_mark_row_dirty` / `grid_mark_all_dirty`).

## Tests

- `tests/parser.tcyr` (70), `tests/grid.tcyr` (76, resize + per-row damage + **multi-word bitset rows≥64**), `tests/unicode.tcyr` (28), `tests/terminal.tcyr` (55, + DECCKM/2004 getters), `tests/render.tcyr` (54 — palette/resolve/paint/glyph/cursor/PPM + **`fb_resize` grow/shrink**), `tests/input.tcyr` (67 — byte-exact + `vt_feed` round-trips + paste), `tests/fbdev.tcyr` (25 — pixel pack + stride/clamp blit vs a fake fb), `tests/evdev.tcyr` (49 — synthetic-event decode + L/R modifiers), `tests/pty.tcyr` (2) + `tests/input_pty.tcyr` (2, real PTY echo — both skip-clean), `tests/puka.tcyr` (2 smoke) — **430 assertions, all green** (`cyrius test`).
- `tests/puka.bcyr` / `tests/puka.fcyr` — bench / fuzz stubs (fuzzing the parser against adversarial input is the M-hardening target).
- **The Wayland subsystem (M6) has no headless tests** — it needs a live compositor, so it is verified by running `programs/puka_term.cyr` on Hyprland (the pure `wire.cyr` codec gets unit tests against captured byte-vectors in M7). The 410 above are the engine + framebuffer cores.

## Carry-forward / known

- **Large static data warning** (~1.2MB): grid backing store (~1.08MB — two `480×144` u64 cell arrays, raised in bite 6 from 132×64) + kashi's font BSS (~99KB — its glyph tables are u64-unit byte arrays). The renderer's pixel buffer + the `wl_shm` present buffer are **heap/memfd-allocated** (grow-only), not static. Acceptable for a desktop binary; heap-allocating the grid remains a deferred optimization (would also let `GRID_MAX_*` grow without BSS cost).
- **Wide CJK glyphs render blank**: kashi's built-in fonts cover CP437 (0x20..0xFF) only, so a width-2 cell paints its background but no glyph until a wider font (PSF/runtime-loaded or `rekha`) lands. The grid/width handling is already correct.
- Deferred (M6 conformance / post-v1.0): DA/DSR query responses, charset designators (ESC ( B), alt-screen (1049), scrollback ring, origin-mode edge cases, mouse tracking, grapheme clustering; non-US keymaps + CapsLock in evdev; AGNOS-native edges (display/input/PTY) are post-v1.0.

## Dependencies

Direct (declared in `cyrius.cyml`):

- stdlib — string, fmt, alloc, io, vec, str, syscalls, assert, bench
- **kashi** — pinned **git dep** (`git`/`tag = "1.0.2"`), **freestanding** `src/font_data.cyr` core only (zero stdlib) — the same core the agnos kernel consumes. Resolves identically on a devbox and in CI (no sibling checkout). Bitmap console glyphs: built-in CP437 VGA 8×16 / CGA 8×8 / VGA 9×16. v1.0.2, API frozen.
- **mabda** — the GPU foundation crate, now a **build dep** (git, tag **3.2.11**); puka uses its **native AMD-GFX9 backend only** (sovereign — the `wgpu` FFI backend is forbidden). Wired in M6 bite 7 via the `pgpu_*` seam (context → render target → readback → `wl_shm`, HW-verified on this Cezanne box). Consuming the amalgam needs the extended `[deps] stdlib` superset (`args/hashmap/tagged/fnptr/mmap/dynlib/sakshi`).

Planned (own-the-stack, not yet wired): `rekha`+`sadish` (vector glyphs, post-v1.0), `sakshi` (logging). kashi's PSF/BDF/PCF loader library face is available but not pulled in (built-in fonts suffice).

## Dep gaps / blockers

- **mabda native RT `va_map` 64 KiB-align bug** *(found 2026-06-19)* — `native_rt_create_2d_rgba8`'s `va_map` returns `EINVAL` unless the BO byte-size is 64 KiB-aligned (256²/512²/1024²/2048² pass; `1260×682×4` = 3360 KiB does not — the GTT BO alloc itself is fine, only the VA map fails). Worked around in `pgpu_target` (pad the alloc to a 256-px multiple per axis, read back the visible sub-rect). The clean fix is mabda-side: round the GTT/`va_map` size up to 64 KiB. Reported to the maintainer.
- **mabda no instanced-vertex path** *(recon 2026-06-19)* — and none is roadmapped (3.2.x closes at 3.2.13; 3.3 = asset-loading). So bite 8 renders the grid via a **single full-screen pass** (texture sampling shipped 3.2.2–3.2.3 + the SPIR-V→GFX9 compiler is complete at 3.2.11, both HW-verified on Cezanne) reading the grid as a storage buffer + the kashi atlas as a texture — NOT instanced quads.
- **mabda dmabuf-export** — mabda has the PRIME `HANDLE_TO_FD` ioctl internally but no public accessor to hand a Wayland client a dmabuf fd (+ fourcc/stride/modifier). Zero-copy GPU present (bite 9 / cut #3) needs a `gpu_render_target_export_dmabuf` in mabda (Phase D, unscheduled). The `wl_shm` GPU→CPU readback memcpy (cut #2, **shipped in bite 7**) needs no mabda change. mabda's native backend is **AMD-GFX9-only** (verified on this Cezanne box) — non-AMD sessions use the `wl_shm` CPU fallback.
- **AGNOS console/desktop environment** — AGNOS can't yet host puka, so **AGNOS-native is post-v1.0** (a new `win_*` backend: `blit`#39 + xHCI/HID + the kernel PTY surface). Not a v1.0 blocker.
- **macOS / X11 / Windows** — `win_*` backends designed-for but not implemented; v1.0 is Wayland-only. macOS needs a Cyrius Darwin backend (cyrius-side).

## Consumers

_None yet._ The v3 command center (post-v1.0) — `thoth` panes — will be the first; it consumes the extracted engine + `aethersafha`.

## Next

**M6 (→ 0.6.x) — finish the Wayland desktop terminal.** The interactive MVP (window +
live shell + correct keyboard + damage-aware render) shipped in 0.6.0; **window resize**
(bite 6) and the **`$SHELL` login-shell + env inheritance** fix landed since. Remaining
bites toward a daily-drivable v1:
1. **mabda GPU** — a kashi glyph atlas + an instanced cell-quad pipeline → a GTT render
   target, presented via `wl_shm` (cut #2), then zero-copy `zwp_linux_dmabuf_v1`
   (cut #3, gated on the mabda export accessor). `wl_shm` stays the permanent CPU fallback.
2. **Retire the framebuffer edges** (bite 10) + a hardening audit of the Wayland wire
   parser (the new untrusted-input boundary).

Then **M7** (conformance: alt-screen, scrollback, charsets, mouse, selection, parser +
wire-parser fuzzing) and **M8** (v1.0 hardening + engine / `aethersafha` extraction).
**AGNOS-native** is post-v1.0; the **command center** is v3. See [`roadmap.md`](roadmap.md).

Resize is a grid dim-change (fixed-max backing), so heap-allocating the grid remains a
deferred optimization — now also the lever for raising `GRID_MAX_*` further without the
~1 MB BSS cost (a >4K or multi-monitor-span window still clamps to 480×144).
