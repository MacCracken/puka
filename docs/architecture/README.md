# Architecture notes

Non-obvious constraints, quirks, and invariants that a reader cannot derive from the code alone. Numbered chronologically — never renumber.

Not decisions (those live in [`../adr/`](../adr/)) and not guides (those live in [`../guides/`](../guides/)). An item here describes *how the world is*, not *what we chose* or *how to do something*.

## Items

- [001 — Dependencies: what each one is for, and what the graph constrains](001-dependencies.md) — every declared dep is compiled into every target; why mabda is not declared; an undeclared `lib/` include resolves from the toolchain; the cyrius 6.5.8 floor; what goes through dhancha and what calls setu directly; what `dist/puka.cyr` consumers must declare; why `path` lines stay commented; no comments inside manifest arrays.

Add the next entry as `002-kebab-case-title.md`. Do not write entries for decisions — those are ADRs.
