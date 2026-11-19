# Project context for AI agents — codebase-intel

> Rules and patterns that apply across the whole repository. This file is a summary; the canonical documents win:
> [prd.md](planning-artifacts/prd.md) · [architecture.md](planning-artifacts/architecture.md) · [eval-spec.md](planning-artifacts/eval-spec.md) · [decisions/](decisions/)

## Product shape (memory aid)

`codebase-intel` = local-first **stdio MCP server** giving AI coding agents AST-aware **hybrid semantic code search** over one repo per index. Differentiator: retrieval quality **measured** against real agent session history, not asserted. Status: spec stage — **no task files until the owner reviews these specs**.

## Technology stack (fixed by ADRs)

- **Rust**, edition 2024; single crate, one binary; library + thin `main.rs`
- **MCP**: `rmcp` — stdio transport only in v1 (ADR 001)
- **Storage**: SQLite, WAL, mmap; **FTS5 trigram** + **sqlite-vec** (bundled extension, in-Rust cosine BLOB fallback) (ADR 002)
- **Chunking**: tree-sitter (rust, typescript/tsx, javascript/jsx, markdown, sql, yaml, toml, json) + gap-span coverage invariant; line-window fallback (ADR 003)
- **Retrieval**: vector top-32 ∪ FTS top-32 → **RRF k=60** → tiny symbol/path boosts; **no reranker** (ADR 004)
- **Embeddings**: Ollama HTTP `localhost:11434`, `nomic-embed-text`, 768-d; `Embedder` trait, circuit-breaker degraded mode (ADR 005)
- **Git**: `gix` (gitoxide) — no shelling out to `git`
- **Hashing**: blake3 · **CLI**: clap · **Runtime**: tokio · **Errors**: thiserror (lib) / anyhow (main only)

## Hard rules

1. **Spec-first.** Any behavior change = spec changes in the same commit. If code and spec disagree, one of them is wrong — fix both.
2. **Failing test first** (TDD). Retrieval-quality tests are the definition of "search works"; extend them when retrieval behavior changes.
3. **No network except localhost Ollama.** CI grep-gate: `reqwest`/`hyper` imports only in `embedder.rs`.
4. **No daemon, no port, no watcher** (ADR 001, ADR 007; full stdio-vs-daemon analysis incl. Postgres/Redis/Docker rejected: ADR 008). Idle cost must be zero.
5. **Every byte covered** — AST chunks + gap spans; the reassembly property test never gets weaker.
6. **The index is disposable.** Code paths may corrupt only "the DB you can rebuild in a minute".
7. **Secret exclusion list lives in `architecture.md` §8** — extend there first, then walker; never the reverse.
8. **Determinism** — no wall-clock, no randomness without seeded RNG in retrieval/eval paths.
9. **No `unwrap()`/`expect()`** in `src/` outside tests and `main.rs`.
10. **`eval` never writes to the index** (module dependency rules, architecture.md §11).

## Performance discipline

- Budgets from PRD §6 (NFR-1..13) are enforced by integration tests where measurable; benchmarks live behind a `bench` feature and are run before any ranking change.
- New code on the query path must justify its cost in the commit message (ms estimate vs budget).
- Anything unbounded (non-prepared statements in loops, per-row transactions, unbounded vectors) is a defect.

## Conventions

- Files `snake_case.rs`; types `PascalCase`; functions `snake_case`; constants `UPPER_SNAKE_CASE`
- Doc comments (`///`) on every public item — this is an open-source project; rustdoc is the user manual until the docs site exists
- Error messages: lowercase, no trailing period, actionable ("`index not built — run `codebase-intel index`"`), never raw SQL/panic dumps
- Conventional commits; **ask before committing**; 20–50 char subject, lowercase, present tense
- Comments explain *why*; rustdoc explains *what*

## Where things live

```
specs/planning-artifacts/prd.md         ← product truth (US/NFR/F IDs)
specs/planning-artifacts/architecture.md ← technical truth (schema, pipeline)
specs/planning-artifacts/eval-spec.md   ← measurement methodology
specs/decisions/NNN-*.md                ← decision history (immutable once Accepted)
specs/tasks/                            ← taskpad (created later, after spec review)
src/                                    ← implementation (once it exists)
tests/fixtures/                         ← synthetic repos + golden queries (no private data)
```

## Dogfood repos

Primary: the author's KanPad (~723 files, ~190 tsx / 161 md / 118 json / 105 rs / 85 ts / 460 commits) — used for real-world tuning. Secondary: taskpad-rs (Rust CLI). Eval fixtures ship from **public** repos only (eval-spec.md §7).
