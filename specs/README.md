# Specs index

> The specs are the source of truth for `codebase-intel`. Code that contradicts a spec is a bug — in the spec or in the code; fix both in the same commit.
>
> Task files (`specs/tasks/`) are managed with `taskpad` and are created **only after these specs are reviewed and approved**.

## Reading order

1. [`planning-artifacts/prd.md`](planning-artifacts/prd.md) — **what & why**: personas, user stories with acceptance criteria, feature list, NFRs with hard budgets, scope.
2. [`planning-artifacts/architecture.md`](planning-artifacts/architecture.md) — **how**: components, SQLite schema, retrieval pipeline, concurrency & performance model, security, testing.
3. [`planning-artifacts/eval-spec.md`](planning-artifacts/eval-spec.md) — **how we know search is good**: session mining, retrieval evaluation, ranking tuning.
4. [`planning-artifacts/epics.md`](planning-artifacts/epics.md) — release phases and epic breakdown (input to the later `taskpad` task breakdown).
5. [`decisions/`](decisions/) — ADRs behind the non-obvious choices (`STATUS / Date / Context / Decision / Consequences / Revisit trigger`).
6. [`project-context.md`](project-context.md) — cross-cutting implementation rules for AI agents.

## Document map

| File | Answers | Owns |
|---|---|---|
| `planning-artifacts/prd.md` | What problem, for whom, what features, what "done" | product truth |
| `planning-artifacts/architecture.md` | Components, data, pipeline, perf, security | technical truth |
| `planning-artifacts/eval-spec.md` | How retrieval quality is measured and tuned | eval truth |
| `planning-artifacts/epics.md` | Phasing: what ships in v0.1 / v0.2 / … | sequencing truth |
| `decisions/NNN-*.md` | Why this, why not that | decision history |
| `project-context.md` | Conventions & guardrails for agents implementing here | editing rules |

## Doc conventions

- PRD uses stable IDs: `US-<epic><nn>` (user stories), `F<nn>` (features), `NFR-<nn>` (non-functional). Acceptance criteria are testable statements ("given/when/then" or imperative checklists) — no "fast", no "intuitive".
- Architecture uses `AST-<nn>` / `ADR-aware` references and links every claim to an ADR where one exists.
- ADRs are numbered monotonically and immutable once Accepted; superseding creates a new ADR.
- **Keep docs small and current.** If an implementation reveals the spec is wrong, the spec changes first, in the same commit as the code.
