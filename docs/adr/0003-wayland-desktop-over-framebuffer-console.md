# 0003 — A Wayland desktop window, not a framebuffer console, is the v1 surface

**Status**: Accepted
**Date**: 2026-06-19

## Context

The platform-agnostic core (M1–M4: parser → grid → terminal → RGB renderer) is
surface-neutral by design. The question is what surface puka actually *runs on* for
v1.

0.5.0 (M5) answered "a Linux framebuffer/VT console": `fbdev.cyr` blitted the RGB
buffer straight to `/dev/fb0`, `evdev.cyr` read raw `/dev/input` events, and
`puka_session.cyr` busy-polled the two. The reasoning was that this is the closest
Linux analogue to the eventual AGNOS native path (own the framebuffer directly,
`blit`#39, no compositor) — a pre-AGNOS proof.

In practice that was the wrong layer for the stated v1 goal — a **daily-drivable
kitty replacement**. On a Linux desktop, everything is a compositor window; a
process that writes `/dev/fb0` fights the compositor for the screen (and on a dev box
`/dev/fb0` *is* the live Hyprland desktop). A framebuffer console is something you'd
reach for `agnoshi`-style — not a terminal you open alongside your other windows. The
references puka studies — Ghostty (a GPU desktop terminal) and supacode (a desktop
command center) — are both desktop-windowed, not console.

## Decision

**v1 puka is a sovereign Wayland client — a real, titled, resizable window in a
Wayland compositor (Hyprland/wlroots) hosting the user's `$SHELL`, GPU-rendered via
`mabda`.** The Wayland wire protocol is spoken directly over the socket (no
libwayland / toolkit / FFI), behind a platform-generic `win_*` window seam destined to
extract to the `aethersafha` windowing crate.

In scope for the pivot (M6): the `win_*` seam (`window.cyr`), the sovereign Wayland
client (`platform/wayland/wire+client+shm`), and a `poll()` event loop. Unchanged and
reused: the M1–M4 core and the `fb.cyr` RGB renderer (it produces the pixels the
`wl_shm` buffer carries) and the evdev **keymap** (lifted to `input/keymap.cyr`).
Superseded and queued for retirement: `fbdev.cyr`, the `evdev` *device* layer, and
`puka_session.cyr`.

AGNOS-native bring-up is **post-v1.0** — AGNOS has no console/desktop environment to
host puka yet, so the AGNOS framebuffer path (`blit`#39 + xHCI/HID + the kernel PTY
surface) becomes a *second* `win_*` backend behind the same seam once the platform is
ready. It was demoted from "the v1 surface" to "the headline post-v1.0 goal", not
abandoned.

## Consequences

- **Positive** — puka is usable on a real Linux desktop today (a window among
  windows), matching the kitty-replacement goal; it composites correctly instead of
  fighting the compositor; it gains GPU rendering via `mabda` and a clean cross-platform
  window seam (X11 / macOS / AGNOS slot in later). The core and renderer carried over
  unchanged, so the pivot cost no engine rework.
- **Negative** — a sovereign Wayland client is a new, substantial untrusted-input
  boundary (compositor messages) to write and harden from scratch; the GPU path is
  gated on `mabda`'s native AMD backend maturing. The AGNOS proof-app slips to
  post-v1.0.
- **Neutral** — `fbdev.cyr` / `evdev` device / `puka_session.cyr` become dead code to
  delete (a roadmap "retire" item, not an emergency); the AGNOS framebuffer work is
  preserved as a future `win_*` backend rather than thrown away.

## Alternatives considered

- **Keep the framebuffer/VT console as v1 (0.5.0's path).** Rejected: it is not a
  desktop terminal — it cannot run as a window alongside the user's session, and the
  daily-driver goal demands exactly that. It also risks clobbering the live desktop.
- **Use a Wayland toolkit / libwayland via FFI.** Rejected: violates the sovereign,
  zero-external-code identity (ADR-0001). The wire protocol is spoken directly.
- **X11 first.** Rejected for v1: the dev/target environment is Hyprland (wlroots); X11
  is a later `win_*` backend, not the v1 path.
