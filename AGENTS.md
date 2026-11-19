# AGENTS.md — codebase-intel

Guidance for AI agents (and humans) working in this repository.

## What this is

`codebase-intel` is a local-first, AST-aware semantic code-search tool for AI
coding agents. It runs as a **stdio MCP server** (spawned on demand by the
client — no daemon), indexes a project's source code into a single SQLite
database (FTS5 + sqlite-vec), and answers hybrid keyword/semantic queries with
compact, line-referenced snippets.

It exists to fix one failure mode: an agent reaches for `grep`/`glob`, misses a
file whose *meaning* matches but whose *spelling* does not, and edits the wrong
thing or duplicates code. See `specs/planning-artifacts/prd.md`.

## How we work: spec-first

The specs are the source of truth. **Do not write implementation code from a
vague task description.** If a task contradicts a spec, stop and fix the spec —
in the same commit as the code.

Reading order before any work:

1. `specs/README.md` — spec index
2. `specs/planning-artifacts/prd.md` — what we're building and why
3. `specs/planning-artifacts/architecture.md` — how it's built
4. `specs/decisions/*.md` — the ADRs behind the non-obvious choices
5. `specs/project-context.md` — cross-cutting rules

Task files (`specs/tasks/Txxx-*.md`) are managed with the `taskpad` CLI and
are **not** created until the specs are reviewed and approved. Do not invent
task files.

## Quality gates (must be green before "done")

- `cargo fmt --check`
- `cargo clippy --all-targets -- -D warnings`
- `cargo test` — unit + integration + retrieval-quality tests; **no test may
  touch the network** (Ollama is mocked in tests)
- `cargo doc --no-deps` — no broken doc links
- If you change behavior, update the spec in the same commit

## House rules

- **TDD.** New behavior starts with a failing test. Retrieval-quality tests
  (golden queries against a fixture repo) are the definition of "search works".
- **No `unwrap()`/`expect()` outside tests and `main.rs`.** Propagate with `?`.
- **No network outside `localhost`** (the Ollama API). No telemetry, no keys.
- **The index is disposable.** Any code path that corrupts it must be
  recoverable with `codebase-intel index --full`.
- **Secret files are never indexed** (`.env*`, `*.pem`, `*.key`, `id_rsa*`,
  `credentials*.json`, …). This list lives in `architecture.md`; extend there,
  not by ad-hoc filtering in code.
- **Commits:** do not commit without being asked. When asked, follow
  conventional commits: `<type>(<scope>): <description>` — lowercase,
  present tense, 20–50 char subject, no trailing period.

## Key paths

- `specs/` — everything spec (see `specs/README.md`)
- `src/` — implementation (once it exists)
- `tests/` — integration & retrieval-quality fixtures
