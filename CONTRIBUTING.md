# Contributing to puka

puka is an AGNOS first-party project. It follows the
[First-Party Standards](https://github.com/MacCracken/agnosticos/blob/main/docs/development/first-party/first-party-standards.md)
and [First-Party Documentation](https://github.com/MacCracken/agnosticos/blob/main/docs/development/first-party/first-party-documentation.md)
standards. Read the genesis repo's
[`CLAUDE.md`](https://github.com/MacCracken/agnosticos/blob/main/CLAUDE.md)
and this repo's [`CLAUDE.md`](CLAUDE.md) before starting.

> **Status: design phase.** The scaffold builds; the VT engine is the next
> milestone. See [`docs/development/roadmap.md`](docs/development/roadmap.md).

## Build and test

```sh
cyrius deps                              # resolve deps
cyrius build src/main.cyr build/puka     # compile
cyrius test                              # run [build].test + tests/*.tcyr
```

The toolchain version is pinned in `cyrius.cyml` (`[package].cyrius`) — that pin
is the single source of truth. Do not hardcode it anywhere else.

## The rule that defines puka

**Sovereign.** puka reimplements the terminal from the VT specs in Cyrius with
**zero external code**. Ghostty and supacode are prior art and conformance
references — never embedded, vendored, or linked. No FFI to Zig/C without an
ADR. See [ADR-0001](docs/adr/0001-sovereign-reimplementation-no-external-engine.md).

Two architectural invariants are non-negotiable: the **parser is pure** (bytes
in, typed actions out — no I/O, no rendering, no hot-path allocation), and the
**grid is the single source of truth** (the renderer is a pure read of it). See
[`docs/architecture/overview.md`](docs/architecture/overview.md).

## Where things go

- **Why we chose X over Y** → a new ADR in [`docs/adr/`](docs/adr/) using [`template.md`](docs/adr/template.md). Zero-padded 4 digits, never renumber, index it.
- **A non-obvious invariant about the code** → a numbered note in [`docs/architecture/`](docs/architecture/) (3 digits, never renumber, indexed).
- **How to do a task** → [`docs/guides/`](docs/guides/).
- **Volatile status** (version, deps, surface) → `docs/development/state.md`.
- **Sequencing** → `docs/development/roadmap.md`.
- **What changed in a version** → [`CHANGELOG.md`](CHANGELOG.md) ([Keep a Changelog](https://keepachangelog.com/) format).

## Conventions

- One change at a time — never bundle unrelated changes.
- Test after every change, not after the feature is "done." Every new escape sequence gets a conformance case.
- Validate all external data (PTY output / file / args); bound every buffer (`var buf[N]` = N **bytes** function-local); never `sys_system()` with unsanitized input — PTY child spawn uses explicit argv.
- Do not modify `lib/` (vendored stdlib / dep symlinks).
- Maintainers handle all git operations and releases.

## Reporting security issues

Do **not** open a public issue for a vulnerability. See [`SECURITY.md`](SECURITY.md).
