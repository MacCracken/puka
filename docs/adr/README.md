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
