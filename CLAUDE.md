# puka — Claude Code Instructions

> **Core rule**: this file is **preferences, process, and procedures** —
> durable rules that change rarely. Volatile state (current version,
> module line counts, supported backends, test counts, dep-gap status,
> consumers) lives in [`docs/development/state.md`](docs/development/state.md).
> Do not inline state here.

## Project Identity

**puka** (Hawaiian: *hole / opening / doorway*) — a sovereign, Cyrius-native
terminal emulator. You open a puka into the machine.

- **Type**: Binary (terminal emulator) — with a VT engine that extracts to a library later (see ADR-0002)
- **License**: GPL-3.0-only
- **Language**: Cyrius (toolchain pinned in `cyrius.cyml [package].cyrius`)
- **Version**: `VERSION` at the project root is the source of truth — do not inline the number here
- **Genesis repo**: [agnosticos](https://github.com/MacCracken/agnosticos)
- **Standards**: [First-Party Standards](https://github.com/MacCracken/agnosticos/blob/main/docs/development/first-party/first-party-standards.md) · [First-Party Documentation](https://github.com/MacCracken/agnosticos/blob/main/docs/development/first-party/first-party-documentation.md)
- **Shared crates**: [shared-crates.md](https://github.com/MacCracken/agnosticos/blob/main/docs/development/planning/shared-crates.md)
- **Naming lane**: Hawaiian (joins `kii`, `hapi`, `anuenue`) — user-facing tool, not a Sanskrit system lib

## Goal

Own the terminal surface for AGNOS: a Ghostty-class emulator — VT/escape-sequence
parser, cell grid, scrollback, font raster, PTY plumbing — written entirely in
Cyrius with **zero external code**. The native AGNOS console, and the eventual
foundation of a worktree coding-agent command center (see ADR-0001, ADR-0002).

## References ARE prior art, NOT dependencies

puka reimplements a solved problem from scratch. The references below are studied
for **architecture and conformance**, never embedded, vendored, or linked:

- **Ghostty / libghostty** (`ghostty-org/ghostty`, Zig) — the architecture reference for a modern GPU terminal.
- **supacode** (`supabitapp/supacode`, Swift/macOS) — the product reference for the phase-2 command center.
- **VT specs** — ECMA-48, DEC STD 070, the Paul Williams DEC ANSI parser state machine, and xterm's `ctlseqs`. These are the conformance target.

If a feature needs "what does Ghostty do here", read the spec and Ghostty's
*approach*, then write Cyrius. **Do not** introduce FFI to Zig/C, do not shell
out to an existing terminal, do not vendor anyone's source.

## Platform reality

Cyrius emits **Linux + agnos** (+ rv64 / bare-metal), **not** macOS/Mach-O.
"Cross-platform" therefore means: dev/test on Linux (the dev host), and native
on the **AGNOS framebuffer** (`blit`#39 — the DOOM/tracker proof-app path, no
compositor needed). The platform-agnostic core (parser + grid) is headless and
unit-testable regardless of backend. A **macOS backend requires a Cyrius Darwin
backend** — that is cyrius-side work and **out of scope for this repo**.

## Current State

> Volatile state lives in [`docs/development/state.md`](docs/development/state.md) —
> current version, surface area, in-flight work, consumers, dep gaps.
> Refreshed every release. Historical release narrative lives in [`CHANGELOG.md`](CHANGELOG.md).

This file (`CLAUDE.md`) is durable rules.

## Scaffolding

Project was scaffolded with `cyrius init puka`. **Do not manually create project
structure** — use the tools. If a tool is missing something, fix the tool.

## Quick Start

```sh
cyrius deps                          # resolve deps
cyrius build src/main.cyr build/puka
cyrius test                          # run [build].test + tests/*.tcyr
```

## Key Principles

- **Correctness over cleverness** — a terminal that mis-renders is a terminal nobody trusts
- **Conformance is measurable** — test against real VT sequences (vttest corpus), not vibes
- **Sovereign** — references are prior art; zero external code, no FFI without an ADR
- Test after every change, not after the feature is "done"
- ONE change at a time — never bundle unrelated changes
- The parser is a **pure state machine** — no I/O, no rendering, no allocation in the hot path; it consumes bytes and emits typed actions
- The **grid is the single source of truth** — the renderer is a pure function of grid state
- **Own the stack**: glyphs via `kashi` (bitmap) → `rekha`+`sadish` (vector); GPU via `mabda`/`ai-hwaccel`; errors/logging via `sakshi`. Never inline what a sibling crate owns
- Build with `cyrius build`, not raw `cat file | cycc` — the manifest auto-resolves deps and prepends includes
- Source files only need project includes — stdlib / external deps auto-resolve from `cyrius.cyml`
- Every buffer declaration is a contract: `var buf[N]` = N **bytes** (function-local) or N×u64 (module-global) — see genesis CLAUDE.md
- Programs call `main()` at top level via a **bare** statement (`_entry();`), exit via `SYS_EXIT` — never `var r = main();`, never literal `60` (agnos correctness; see genesis CLAUDE.md)

## Rules (Hard Constraints)

- **Read the genesis repo's CLAUDE.md first** — [agnosticos/CLAUDE.md](https://github.com/MacCracken/agnosticos/blob/main/CLAUDE.md)
- **Do not commit or push** — the user handles all git operations
- **Never use `gh` CLI** — use `curl` to the GitHub API if needed
- **No external code** — no embedding/vendoring/linking libghostty or any non-AGNOS source; no FFI to Zig/C without an ADR
- Do not skip tests before claiming changes work
- Do not use `sys_system()` with unsanitized input — command injection. PTY child spawn uses explicit argv (`exec_vec`), never a shell string
- Do not trust external data (PTY output, file, network, args) without validation — the parser eats **adversarial** byte streams by definition
- Do not modify `lib/` files (vendored stdlib / dep symlinks)
- Do not hardcode toolchain versions in CI YAML — `cyrius = "X.Y.Z"` in `cyrius.cyml` is the source of truth
- Do not assume threads — AGNOS is single-threaded/single-core today (the read-loop + render must not rely on concurrency; see the multithreading-arc note in genesis)

## Documentation

- [`docs/adr/`](docs/adr/) — Architecture Decision Records (*why X over Y?*)
- [`docs/architecture/`](docs/architecture/) — Non-obvious constraints (*what's true about the code?*) + [`overview.md`](docs/architecture/overview.md) (module map + data flow)
- [`docs/guides/`](docs/guides/) — Task-oriented how-tos
- [`docs/examples/`](docs/examples/) — Runnable examples
- [`docs/development/state.md`](docs/development/state.md) — Live state snapshot
- [`docs/development/roadmap.md`](docs/development/roadmap.md) — Milestones through v1.0

## Process

### Work Loop (continuous)

1. **Work phase** — features, roadmap items, bug fixes (small bites — terminal work is large-effort)
2. **Build check** — `cyrius build`
3. **Test + benchmark additions** for new code — conformance cases for every new escape sequence
4. **Internal review** — performance, memory, correctness, edge cases (malformed/truncated sequences)
5. **Security check** — any new syscall usage, PTY/input handling, buffer allocation
6. **Documentation** — update CHANGELOG, `docs/development/state.md`, any ADR/architecture note the change earned
7. **Version sync** — `VERSION`, `cyrius.cyml`, CHANGELOG header
8. **Return to step 1**

### P(-1): Hardening (before features on a fresh scaffold, before each minor, before v1.0)

Test+bench sweep → fmt/lint/vet clean → baseline benchmarks → audit (input validation, buffer safety, syscall review, no command injection) → file findings in `docs/audit/YYYY-MM-DD-audit.md` → doc audit. The parser is the highest-priority audit target — it is the untrusted-input boundary.

### Task Sizing

- **Low/Medium**: batch freely. **Large** (which most terminal work is): small bites, verify each before the next. **If unsure**: treat as large.
