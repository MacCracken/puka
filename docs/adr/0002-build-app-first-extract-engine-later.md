# 0002 — Build the terminal app first; extract the VT engine to a library later

**Status**: Accepted
**Date**: 2026-06-18

## Context

puka has two horizons. Near-term ("2", in the user's framing) it is a terminal
emulator. Long-term ("1") it is the foundation of a worktree coding-agent
command center (a sovereign supacode-equivalent): multiple panes, one terminal
surface each, every pane running a `thoth` agent session in its own git
worktree.

That long-term shape is exactly the Ghostty split: `libghostty` (the embeddable
engine) and an application that embeds it. So puka's VT engine (parser, grid,
terminal semantics, Unicode) will eventually have **two** consumers: the puka
terminal itself, and the command center.

The question: build the engine as a standalone library *now* (designing its API
up front for two consumers), or build the terminal app first and extract the
engine *when the second consumer actually exists*.

## Decision

**Build puka as a terminal-emulator binary first. Keep the VT engine as a clean
internal module boundary, but do not extract it to a separate library until the
phase-2 command center exists as a real second consumer.** At extraction, the
engine becomes a Sanskrit-named substrate library (system-lib naming lane);
puka becomes a thin consumer of it.

This follows the established AGNOS extraction discipline: `yo`→`taar`,
`iam`→`mihi`, `darshini`→`darshana` — the second concrete consumer's needs shape
the extracted API more honestly than upfront design around hypothetical
consumers.

## Consequences

- **Positive** — the engine API is designed against two *real* call sites, not one real and one imagined; avoids the "designed around hypotheticals" anti-pattern.
- **Positive** — puka ships a working terminal sooner; no library-packaging overhead during the milestones where there's only one consumer.
- **Positive** — naming-lane discipline is preserved: puka (Hawaiian, user-facing) now; the engine lib (Sanskrit, system substrate) at extraction.
- **Negative** — requires discipline to keep the engine boundary clean inside the binary so extraction is mechanical rather than a rewrite. The roadmap's v1.0 criterion "engine API frozen" is the checkpoint that makes this real.
- **Neutral** — the engine-lib name is deferred to extraction time (consistent with `taar`/`mihi` being named/carved at second-consumer time).

## Alternatives considered

- **Build the engine as a library from day one** — rejected. Only one consumer exists (puka itself); designing a public API around a future, unbuilt command center repeats the over-abstraction the ecosystem explicitly avoids. The internal-boundary approach gets the same modularity without committing a public surface early.
- **Never extract — keep one monolithic repo** — rejected. It would force the command center to depend on the whole terminal binary rather than the engine, violating the monolithic-by-design principle (subsystems are separately-releasable; coupling is at the ABI/contract, never the codebase).
