# Architecture Decision Records

Decisions about puka — what we chose, the context, and the consequences we accept. Use these when a future reader would reasonably ask *"why did we do it this way?"*

## Conventions

- **Filename**: `NNNN-kebab-case-title.md`, zero-padded to four digits. Never renumber.
- **One decision per ADR.** If a decision supersedes a prior one, add a new ADR and set the old one's status to `Superseded by NNNN`.
- **Status lifecycle**: `Proposed` → `Accepted` → (optionally) `Superseded` or `Deprecated`.
- Use [`template.md`](template.md) as the starting point.

## ADR vs. architecture note vs. guide

| Kind | Lives in | Answers |
|---|---|---|
| ADR | `docs/adr/` | *Why did we choose X over Y?* |
| Architecture note | `docs/architecture/` | *What non-obvious constraint is true about the code?* |
| Guide | `docs/guides/` | *How do I do X?* |

## Index

- [0001 — Sovereign reimplementation; no libghostty, no external engine](0001-sovereign-reimplementation-no-external-engine.md) — reimplement the terminal in Cyrius from the VT specs; references are prior art, never embedded code.
- [0002 — Build the terminal app first; extract the VT engine to a library later](0002-build-app-first-extract-engine-later.md) — app-first, engine extracted to a Sanskrit-named lib when the phase-2 command center becomes the second consumer.
- [0003 — A Wayland desktop window, not a framebuffer console, is the v1 surface](0003-wayland-desktop-over-framebuffer-console.md) — 0.6.0 pivot: v1 is a sovereign Wayland client (a window in Hyprland hosting `$SHELL`); the 0.5.0 `/dev/fb0` console (`fbdev`/`evdev`/`puka_session`) is superseded; AGNOS-native framebuffer is a post-v1.0 `win_*` backend.
