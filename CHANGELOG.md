# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added
- **M1 Bite 1 — VT parser** (`src/parser.cyr`): the Paul Williams DEC ANSI parser state machine, pure (bytes in → one typed event out via `vt_feed`, no I/O / no rendering / no hot-path allocation). Handles GROUND/ESCAPE/CSI/OSC fully; DCS + SOS/PM/APC consumed-and-discarded (full DCS passthrough is M6). Events: PRINT / EXECUTE / CSI / ESC / OSC, with `vt_param*` / `vt_prefix` / `vt_intermediate` / `vt_string_*` accessors. UTF-8-mode (bytes ≥0x80 printable; no 8-bit C1). `tests/parser.tcyr` — 70 assertions green (CUP, defaults, DECSET private markers, SGR incl. `;`/`:` truecolor, ESC dispatch + charset intermediates, OSC BEL/ST termination, CSI-ignore recovery, CAN abort, mid-sequence control execution, param clamp/cap).
- Design docs: filled `CLAUDE.md` (puka identity, sovereign principles, platform reality), `docs/development/roadmap.md` (milestones M1–M7 + v1.0 criteria + phase-2 command center), `docs/architecture/overview.md` (module map + data flow), ADR-0001 (sovereign reimplementation, no libghostty), ADR-0002 (app-first, engine extracted later), `docs/development/state.md`, README.
- Root docs the scaffolder skipped: `CONTRIBUTING.md`, `SECURITY.md`, `CODE_OF_CONDUCT.md`.

## [0.1.0]

### Added
- Initial project scaffold (`cyrius init puka`) — hello-world builds and runs on the Linux host; cyrius pin `6.2.21`.
