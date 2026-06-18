# 0001 — Sovereign reimplementation; no libghostty, no external engine

**Status**: Accepted
**Date**: 2026-06-18

## Context

The project began from two references: **Ghostty** (`ghostty-org/ghostty`, Zig)
— a fast, modern terminal emulator whose engine is factored into the embeddable
`libghostty` — and **supacode** (`supabitapp/supacode`, Swift/macOS), a
"worktree coding-agents command center" that embeds libghostty for its terminal
surfaces. The obvious shortcut is what supacode did: embed libghostty and build
the product on top.

AGNOS is a sovereign OS built in Cyrius, a language with zero external
dependencies. Cyrius emits Linux + agnos (+ rv64 / bare-metal), not macOS. A
terminal emulator is a core piece of the AGNOS console surface and must run
natively on the AGNOS framebuffer, on iron, with no foreign runtime.

The decision: embed an existing engine, or reimplement.

## Decision

**Reimplement the terminal emulator from scratch in Cyrius. Depend on zero
external code.** Ghostty (architecture), supacode (product shape), and the VT
specifications — ECMA-48, DEC STD 070, the Paul Williams DEC ANSI parser state
machine, xterm `ctlseqs` — are **prior art and conformance targets**, studied
and referenced, never embedded, vendored, or linked.

In scope: a Cyrius implementation of the VT parser, cell-grid model, terminal
semantics, PTY plumbing, input encoding, and framebuffer rendering. Out of
scope: any FFI to Zig/C, any vendored source, any shell-out to an existing
terminal. Introducing FFI later requires a superseding ADR.

## Consequences

- **Positive** — full sovereignty; runs natively on the AGNOS framebuffer with no foreign runtime; no Zig/C toolchain or FFI surface; the codebase is auditable end-to-end in one language; conformance is owned and measurable against the specs.
- **Positive** — the VT engine can later extract to an AGNOS substrate library (ADR-0002) because we own every line.
- **Negative** — we re-own a large, conformance-heavy surface: the full escape-sequence grammar, Unicode width, PTY semantics, font raster, and decades of xterm edge cases that libghostty already handles. This is a multi-milestone effort (see the roadmap).
- **Negative** — no free ride on Ghostty's GPU renderer or its battle-tested conformance; we must build and test our own.
- **Neutral** — Ghostty/supacode remain living references; we track their *approaches* without taking their *code*.

## Alternatives considered

- **Embed libghostty (the supacode path)** — rejected. Requires Zig FFI, ships a foreign runtime, cannot run on agnos (no Cyrius Darwin/Zig bridge), and violates the sovereignty constraint. The user's framing was explicit: build it in Cyrius, "not using another person's code."
- **Thin wrapper / config layer over an existing terminal (e.g. ship Ghostty, theme it)** — rejected for the same reasons, plus it wouldn't run on AGNOS at all.
- **Port libghostty's source to Cyrius mechanically** — rejected. A line-by-line transliteration from Zig inherits Zig idioms and isn't meaningfully sovereign; redesigning from the *specs* (with Ghostty as one reference of several) yields a Cyrius-native design per the ecosystem's "redesign, don't reinvent" practice.
