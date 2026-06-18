# Security Policy

> puka is a **terminal emulator**: it parses an untrusted byte stream (whatever
> a child process writes) and spawns child processes over a PTY. The VT parser
> is the load-bearing security boundary — it eats adversarial input by
> definition. See [ADR-0001](docs/adr/0001-sovereign-reimplementation-no-external-engine.md)
> and the parser-purity invariant in [`docs/architecture/overview.md`](docs/architecture/overview.md).

## Supported versions

puka is **pre-release**. No tagged release exists yet, so there is no
supported-version matrix. Until the first release, only the tip of the default
branch is in scope.

| Version | Supported |
| ------- | --------- |
| `main` (unreleased) | ✅ best-effort |
| any tagged release  | — none yet |

## The AGNOS security model

puka does **not** own its own security boundary. In AGNOS:

- **kavach** owns the sandbox — applications never manage their own security boundaries.
- **t-ron** owns MCP per-tool authorization (relevant to the phase-2 command center, not the terminal core).

puka **consumes** these; it does not reimplement them. Within puka's own scope,
the discipline is: the VT parser validates and bounds everything it reads; PTY
child processes are spawned with explicit argv (`exec_vec`), never a shell
string; every buffer is bounded and every syscall return is checked.

## Reporting a vulnerability

Please report suspected vulnerabilities **privately** — do not open a public
issue for an unfixed flaw.

- Email: **cyriusmaccken@gmail.com** with a subject line starting `puka security:`
- Include: affected component, reproduction steps, impact, and any suggested fix.
- You will get an acknowledgement as soon as is practical. Coordinated
  disclosure is appreciated; please give a reasonable window before public
  disclosure.

Fixes are tracked per the AGNOS first-party process; CVE-severity findings are
recorded under `docs/audit/` once the security-audit pass exists.

## Scope notes

- **puka-specific** issues — the VT parser, grid/terminal state, PTY plumbing, input encoding, the framebuffer renderer — belong here.
- Issues in a consumed crate (`kashi`, `rekha`, `mabda`, `sakshi`, kavach, t-ron) belong to the owning project — report them there.
