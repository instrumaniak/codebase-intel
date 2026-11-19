# Product Requirements Document — codebase-intel

**Status:** v1 draft — awaiting owner review before any task breakdown
**Audience:** maintainer (Raziur), AI agents implementing the project
**Companion docs:** [architecture.md](architecture.md) · [eval-spec.md](eval-spec.md) · [epics.md](epics.md) · [../decisions/](../decisions/)

---

## 1. Overview

`codebase-intel` is a **local-first, AST-aware semantic code-search MCP server** for AI coding agents. It indexes a repository into a single on-disk SQLite database and serves hybrid (vector + keyword) retrieval over that index to any MCP client (opencode first). It is built because an agent that only has `grep` and `glob` systematically misses files whose *meaning* matches the task but whose *spelling* does not, and consequently edits the wrong file, duplicates existing code, or reads far too many files to be safe.

Retrieval quality is not asserted — it is **measured**, using a purpose-built evaluation harness that derives ground truth from real agent session history (`eval-spec.md`). That measurement loop is the project's differentiator: existing semantic-search MCP servers ship fixed ranking; this project treats ranking as something to be tuned against evidence.

### What this is NOT

- Not a chat assistant, code generator, or refactoring bot
- Not a hosted service, not a SaaS, no account, no telemetry
- Not a knowledge base of conversations (session search is *input to evaluation*, not a feature of v1)
- Not a replacement for `grep`/`glob` — it complements them
- Not multi-repo in v1 (one project per index)

## 2. Personas

| ID | Persona | Description | Primary need |
|---|---|---|---|
| P1 | **The coding agent** (opencode et al.) | Calls MCP tools as tools; context window is scarce; retries anything that fails | Fast, compact, accurate retrieval; compact snippets with file:line refs; graceful degradation |
| P2 | **The developer** (repo owner) | Runs the tool on their own machines while developing; is also the maintainer | Zero-config start; invisible in normal dev; no interference with builds/tests |
| P3 | **The open-source user** | Clones the repo, `cargo install`, wires it into their agent of choice | One-command install; docs; works offline; sane defaults |
| P4 | *(deferred)* **The team** | Shared index across machines/team | Out of scope until v1 proves single-user value |

P1 is the design driver for every interface decision: tools must be cheap to call, results must be cheap to consume, instructions must teach the agent when to prefer `code_search` over `grep`.

## 3. Goals and non-goals

### 3.1 Goals

- G1. An agent working in a 1k-file repo can find the right handful of files for a natural-language task in one tool call
- G2. Indexing and querying are invisible in cost and interference to normal development (builds, tests, editors)
- G3. Retrieval quality is measurable; every ranking change is benchmarked before/after
- G4. Runs fully offline except a localhost Ollama call for query embeddings; degrades gracefully to keyword-only when Ollama is down
- G5. Open-source it cleanly (license, CI, release binaries, docs) once v0.1 proves itself

### 3.2 Non-goals (v1)

- Code generation, editing, or application
- Session/chat-history search as a *user-facing* feature
- Re-ranking with cross-encoders (see ADR 004 — explicitly rejected for v1)
- Filesystem watching / live updates (reindex is lazy/manifest-based in v1)
- Multi-project indexes in one DB, remote/server mode, web UI
- Windows support (Linux first; macOS best-effort; Windows unsupported until demanded)
- Secret scanning — we *exclude* likely-secret files; we do not scan for secrets

## 4. User stories

Grouped by epic. IDs are stable; acceptance criteria are testable. "AC" = acceptance criterion.

### E1 — Indexer core

**US-101 — Index a repo without asking questions**
> As the developer (P2), I run `codebase-intel index` in any git repo and the index is built respecting `.gitignore`, without me configuring anything else.

- AC1: Honors `.gitignore`, `.git/info/exclude`, and nested ignore files
- AC2: Skips: files > 256 KB, binary files, lockfiles (`package-lock.json`, `yarn.lock`, `Cargo.lock`, `pnpm-lock.yaml`, `bun.lockb`), minified assets (avg line length > 2 KB), and the built-in secret-exclusion list (`.env*`, `*.pem`, `*.key`, `*.p12`, `id_rsa*`, `id_ed25519*`, `credentials*.json`, `*.keystore`, `*.jks`, `*.pfx`)
- AC3: Creates zero files inside the repo (index lives in `~/.local/share/codebase-intel/`)
- AC4: Idempotent — running twice in a row re-indexes nothing when the tree is unchanged

**US-102 — Incremental reindex**
> As the developer, after I edit a few files, `codebase-intel index` re-embeds only those files.

- AC1: Change detection via content hash (blake3) of every walked file vs manifest in `files` table
- AC2: Changed file → its chunks (and their FTS/vec rows) are deleted and re-created atomically in one transaction
- AC3: Deleted file → its rows are removed; renamed file → treated as delete + add
- AC4: A no-change run completes in < 1 s on the 1k-file target repo

**US-103 — AST-aware chunking with complete coverage**
> As the agent (P1), search results always include the definition of what I need — never a half-function.

- AC1: For supported languages, chunks are tree-sitter AST nodes (functions, methods, impl/trait blocks, classes, structs, enums, type aliases, exported consts) with doc comments/docstrings attached
- AC2: Every byte of indexed text belongs to exactly one chunk — region filling ("gap spans") between symbols is indexed line-based so nothing is invisible to search
- AC3: Chunk text includes: file path header, parent symbol breadcrumb, doc comment, body (truncated at embedder budget with explicit marker, never silently)
- AC4: Non-supported languages fall back to line-window chunks with the same coverage guarantee

**US-104 — Know when the index is stale**
> As the agent, `index_status` tells me if results may be out of date before I trust them.

- AC1: `index_status` reports: schema version, project root, index creation/update time, full/incremental state, git HEAD at index time vs current HEAD, count of indexed files/chunks, embedder model + availability, estimated stale-file count
- AC2: `code_search` results include a one-line staleness note when the index is known-stale (e.g., HEAD moved or working-tree hash differs from manifest), never silently stale

**US-105 — Degrade gracefully when Ollama is down**
> As the agent, if Ollama isn't running, search still works via keyword (FTS) matching and tells me it's degraded.

- AC1: On Ollama connect failure/timeout, falls back to FTS-only results and marks them `degraded: true` in output JSON
- AC2: No crash; the warning is visible in `index_status` and in the search result footer
- AC3: Recovery is automatic on the next call (embedder health re-probed per call, cached ≤ 30 s)

### E2 — Retrieval

**US-201 — Semantic code search**
> As the agent, I describe the behavior I need ("the code that refreshes expiring auth tokens") and get the right files back, even if none of my query terms appear verbatim.

- AC1: `code_search(query, limit?, path_prefix?, kind?, lang?)` returns ranked results: `path`, `start_line`, `end_line`, `symbol` (when known), `kind`, fused score, snippet (≤ 20 lines)
- AC2: Hybrid retrieval: vector top-32 (sqlite-vec, cosine) ∪ FTS5/BM25 top-32, fused with RRF (k=60); see [ADR 004](../decisions/004-rrf-hybrid-retrieval-no-reranker-v1.md)
- AC3: Symbol-name boosting: exact/prefix symbol-name match in query raises that chunk's rank (exact ×1.5, prefix ×1.25, case-insensitive)
- AC4: `limit` default 10, max 25; output stays compact — full text only via `get_file_outline` or the client's own `read`

**US-202 — Exact identifier lookup stays reliable**
> As the agent, `code_search("handleJwtRefresh")` finds the function even if embeddings are weak on identifiers.

- AC1: FTS5 with trigram tokenization handles substrings/identifiers (≥ 3 chars) — `JwtRefr` matches `handleJwtRefresh`
- AC2: Exact-identifier queries (regex `^[A-Za-z_][A-Za-z0-9_]*$`) are allowed to short-circuit to FTS-only ranking when the identifier exists in the index
- AC3: Keyword-only mode performs within 20% of full hybrid on the eval set for identifier queries (verified by `eval`, not assumed)

**US-203 — Filter results**
> As the agent, when I know I want Rust source or a specific directory, I scope the search.

- AC1: `path_prefix` matches repo-relative path prefixes, normalized (leading `./` stripped, `/` separators)
- AC2: `kind` filters by chunk kind (`function`, `method`, `class`, `struct`, `impl`, `module`, `doc`, `other`)
- AC3: `lang` filters by language id
- AC4: Filters are applied at the SQL layer pre-fusion (not post-hoc), so filtered searches return real top-N rankings within the filter

**US-204 — Compact, token-budgeted output**
> As the agent, results fit my working context: no file bodies bigger than the snippet needs.

- AC1: Default result set ≤ ~2,000 tokens (configurable via `limit` only, not by raw byte dump)
- AC2: Every result carries repo-relative path + line span so the client can `read` precisely the lines it needs
- AC3: JSON output is stable and documented (schema in architecture.md §6)

**US-205 — Outline of a file**
> As the agent, `get_file_outline(path)` lists what's in a file without reading it.

- AC1: Returns symbols with kind, name, parent, line range, doc-comment first line
- AC2: Works for files that were not indexed (clear error `file_not_indexed`, suggests `reindex`)
- AC3: Unknown-language file → returns top-level structure if linearly chunked, else error with reason

### E3 — MCP surface & server lifecycle

**US-301 — Zero-config MCP launch**
> As the developer, I add one entry to my MCP config and opencode spawns the server on demand.

- AC1: Server speaks MCP over stdio (rmcp); tools: `code_search`, `get_file_outline`, `index_status`, `reindex`
- AC2: Server instructions are emitted at handshake directing agents to prefer `code_search` before `grep` for conceptual queries, and to follow up with `read` for exact code
- AC3: First query on a cold server (no index) performs a full index inline and answers the query in the same call, with timing reported
- AC4: `reindex` tool: `mode: "incremental" (default) | "full"`; returns stats (files processed, chunks written, duration)

**US-302 — Concurrent with active development**
> As the developer, my builds and tests run at full speed while the indexer works, worst case the indexer slows itself down.

- AC1: Indexing uses a worker thread with reduced scheduling priority (`nice(19)` on Unix; best-effort on macOS); it never holds a lock needed by the query path
- AC2: Reads (queries) use WAL snapshot reads; indexing writes transactions are ≤ 200 ms each (per-file batching)
- AC3: SQLITE_MMAP_SIZE is set so read-heavy pages come from page cache, not syscalls
- AC4: If two `serve` processes are spawned (two opencode windows), one is read-only anyway during normal operation, and concurrent reindex attempts serialize on a file lock (`O_EXLOCK` on `<db>.lock`) — loser skips reindex, reports `busy` in status

**US-303 — Honest resource appetite**
> As the developer, the tool is invisible in daily use.

- AC1: Warm query ≤ 150 ms p50, ≤ 400 ms p95 excluding Ollama network call to a *running* model (query embedding time is reported separately in `index_status`)
- AC2: RSS ≤ 100 MB steady state for a 1k-file repo (excluding Ollama's own memory)
- AC3: Idle server (no queries) uses 0% CPU indefinitely
- AC4: Full index of a 1k-file repo (~3k chunks) ≤ 60 s on a mid-range 2023 laptop CPU, embedding 768-dim vectors at ≥ 20 chunks/s via Ollama

### E4 — Evaluation (Phase 2, v0.2) — see eval-spec.md

**US-401 — Mine agent session history offline**
> As the maintainer, I snapshot opencode's DB and extract (task → files actually edited/read) pairs without touching the live DB.

- AC1: `mine-sessions --db <snapshot> --project <path>` works on a `.backup` copy; the live DB is never opened
- AC2: Output: JSONL records `{session_id, turn, task_text, files_read[], files_edited[], greps[], globs[], zero_hit_searches[]}`
- AC3: No session text is committed to git without sanitization; snapshot files are deleted after mining by default (`--keep-snapshot` to override)

**US-402 — Derive ground truth**
> As the maintainer, I have a labeled eval set built from real agent behavior in this repo family (KanPad, taskpad-rs).

- AC1: Labels: `edited` (strong positive) > `read` (medium) > `grep-hit-not-read` (weak); also negative examples from zero-hit searches
- AC2: Eval set deduplicated by task-text similarity (trigram Jaccard ≥ 0.8 ⇒ drop)
- AC3: For the public repo, fixtures are generated by running agents against public repos, not from private session text; private eval sets never leave the machine

**US-403 — Measure retrieval quality**
> As the maintainer, `eval` reports Recall@1/5/10, MRR, nDCG@10 of the retriever against the mined set, overall and per query-type bucket.

- AC1: Deterministic — same index + same eval set ⇒ identical metrics (seed fixed for tie-breaking)
- AC2: Report includes per-bucket breakdown: identifiers / error messages / conceptual behavior / config-refactor
- AC3: Baseline captured (grep-only, dense-only, FTS-only, hybrid RRF) before any tuning is attempted

**US-404 — Tune what the eval harness can measure**
> As the maintainer, I adjust chunking, RRF k, and boost weights and keep only changes that improve the metrics.

- AC1: Every accepted ranking change ships with before/after eval numbers in the commit message or PR
- AC2: Regressions > 2% on Recall@10 block merge (CI retrieval-quality gate once fixtures exist)
- AC3: Final tuned weights documented in architecture.md §5 with the eval data that produced them

### E5 — Git intelligence (Phase 3, v0.3)

**US-501 — Semantic commit search**
> As the agent, `git_search("migrate auth to refresh tokens")` returns the commits that did it and the files they touched.

- AC1: Indexes commit hash, author date, subject, body, and file paths touched (embeddings over subject+body)
- AC2: Results link commits → files; file results can be scoped with `since`/`until`
- AC3: Merge commits and bot commits are excluded from the embedding index (still usable for churn stats)
- AC4: Churn metadata (times-touched-last-90-days per file) feeds a bounded recency boost (≤ 5% of fused score)

### E6 — OSS release (Phase 4, v0.4)

**US-601 — Installable artifact**
> As a stranger, I can install and use the tool without talking to the author.

- AC1: `cargo install codebase-intel` and prebuilt GitHub release binaries (Linux x86_64/aarch64)
- AC2: README: install, opencode/Claude Code config snippets, troubleshooting, perf numbers
- AC3: CI: fmt, clippy, tests (mock-Ollama), docs build; release workflow via cargo-dist
- AC4: CONTRIBUTING.md + dual LICENSE-MIT/APACHE (already in repo)

## 5. Features

| ID | Feature | Epic | Phase | Depends on |
|---|---|---|---|---|
| F1 | Gitignore-aware walker + size/type filters | E1 | v0.1 | — |
| F2 | tree-sitter chunking (rust, ts/tsx, js/jsx, md, sql, yaml/toml/json) + line-window fallback + gap-span coverage | E1 | v0.1 | F1 |
| F3 | SQLite store: schema, WAL, mmap, transactions, manifest | E1 | v0.1 | — |
| F4 | Ollama embedder (batch, retry, circuit-break, FTS fallback) | E1 | v0.1 | — |
| F5 | Hybrid retrieval (vector + FTS5 → RRF) + symbol boost + filters | E2 | v0.1 | F2–F4 |
| F6 | MCP stdio server: tools + instructions + staleness note | E3 | v0.1 | F5 |
| F7 | `get_file_outline`, `index_status`, `reindex` | E3 | v0.1 | F6 |
| F8 | Session miner (offline snapshot → JSONL) | E4 | v0.2 | — |
| F9 | Eval harness + golden fixtures + baseline metrics report | E4 | v0.2 | F5, F8 |
| F10 | Ranking tuning loop (documented outcomes) | E4 | v0.2 | F9 |
| F11 | Git log indexing + `git_search` + churn boost | E5 | v0.3 | F5 |
| F12 | OSS release: CI, releases, docs, packaging | E6 | v0.4 | F1–F11 |

File-level cut for v0.1 (F1–F7): walker, chunker, store, embedder, retrieval, MCP tools. No eval, no git, no watcher.

## 6. Non-functional requirements

Budgets are **hard targets**, enforced by tests/benchmarks where measurable. "Warm" = server process already has the index open.

| ID | Requirement | Budget | Verification |
|---|---|---|---|
| NFR-1 | Warm query latency, p50 / p95 (excluding embedder network call) | ≤ 150 / ≤ 400 ms | Criterion bench + CI timing log |
| NFR-2 | Warm query latency including local Ollama embedding (nomic-embed-text) | ≤ 600 ms p95 | Integration test with live Ollama (feature-gated) |
| NFR-3 | Cold start → first query answered (index exists) | ≤ 500 ms | Integration test |
| NFR-4 | RSS steady state, 1k-file repo | ≤ 100 MB | Integration test with `/proc/self/status` sampling |
| NFR-5 | Full index throughput (embedded) | ≥ 20 chunks/s; 3k chunks ≤ 60 s | Integration test + manual bench |
| NFR-6 | No-change incremental index | ≤ 1 s | Integration test |
| NFR-7 | Idle CPU | 0% | Manual; enforced by "no background threads in idle path" design rule |
| NFR-8 | Network | localhost:11434 only; no telemetry; no keys | Code review + CI grep gate |
| NFR-9 | Test suite | no network; mocked Ollama; ≤ 60 s total | CI |
| NFR-10 | Index DB size, 1k-file repo | ≤ 150 MB | Informational |
| NFR-11 | Determinism | identical results for identical query+index state | Unit tests on RRF/fusion |
| NFR-12 | Portability | Linux first-class; macOS builds and tests pass; Windows: unsupported, documented | CI (linux + macos) |
| NFR-13 | Recovery | `index --full` rebuilds from scratch; a crashed index never needs manual surgery | Chaos-ish integration test (kill mid-index → status → reindex --full → green) |

## 7. Scope

### 7.1 In scope, v0.1 (MVP)

F1–F7. Single repository per index. Rust binary, Linux. Ollama embedder with FTS-only fallback. MCP tools as listed. Manual reindex (`index`, `reindex`).

### 7.2 In scope, v0.2

F8–F10: offline session mining, eval harness, golden fixtures from public/sanitized data, documented tuning results.

### 7.3 In scope, v0.3 / v0.4

F11 (git intelligence), F12 (OSS release).

### 7.4 Explicitly out of scope

See §3.2. Additionally: migrating between DB schema versions (v1 ships schema v1; migrations arrive when schema changes after release), multi-client synchronization, encryption at rest (the index contains no secrets by construction — the exclusion list — but the DB file is not encrypted).

## 8. Success criteria

| ID | Criterion | Measured how |
|---|---|---|
| S1 | In KanPad (dogfood repo), agent finds the right file for 9/10 conceptual queries | Manual 10-task spot check; results in eval report |
| S2 | Eval baseline documented before tuning (Recall@10, hybrid vs dense-only vs FTS-only) | `eval --baseline` output committed to `docs/eval/` |
| S3 | Post-tuning Recall@10 improves ≥ 5% over baseline on holdout set, or the change is rejected | eval-spec.md §6 |
| S4 | `grep` remains a tool the agent reaches for — `code_search` must be *added*, not mandated | MCP instructions quality check |
| S5 | Zero incidents of index corrupting the working tree or eating unbounded memory in dogfooding for 30 days | Dogfood log |
| S6 | Published release with working `cargo install` | GitHub releases |

## 9. Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Embedder quality (nomic-embed-text) is not good enough for code semantics | Medium | High | Eval harness exists precisely for this; swap embedder via `Embedder` trait; try bge-code-v1 / qwen3-embedding if numbers disappoint (ADR 005 documents alternatives) |
| tree-sitter grammar coverage for TS/MD/SQL costs more than expected | Medium | Medium | v1 languages ordered by dogfood repo mix; line-window fallback guarantees coverage regardless; stub grammars behind feature flags |
| Ollama absent/slow on user machines | Medium | Low | FTS-only degraded mode is a first-class design path, tested |
| sqlite-vec native extension packaging friction | Low | Medium | Bundled via `sqlite-vec` crate's `bundled` feature; fallback: in-Rust cosine over BLOB column if extension fails to load |
| Agent context bloat from too many/long results | Low | Medium | Hard caps (20-line snippets, ≤ 25 results, ≤ 2k tokens default); measured in eval |
| Privacy mistakes when open-sourcing eval fixtures | Low | High | Fixtures from public repos only; CI secret-scan on fixtures; session snapshots never committed |
| Name collision at publish time | Medium | Low | Rename before v0.4; crates.io/npm/GitHub recheck then |

## 10. Open questions

| ID | Question | Current leaning | Blocks |
|---|---|---|---|
| Q1 | Crate name stays `codebase-intel`? | Yes — free on crates.io & GitHub check pending re-verify at release | v0.4 |
| Q2 | Do we ship an npm wrapper (`npx codebase-intel`)? | Maybe in v0.4; Rust install is primary | v0.4 |
| Q3 | macOS CI — worth the runner cost pre-release? | Yes (it's core, not "extra") | v0.1 |
| Q4 | Index poisoned by `.gitignore`d-but-important files (e.g., `dist/` fixtures)? | Also honor `.git/info/exclude`; document `.codebase-intel-include` escape hatch | v0.1 design review |
| Q5 | Should `reindex` also embed *queries agents actually made* to pre-warm nothing? | No — overfitting to history; the eval loop is offline | n/a |

## 11. Release plan summary

| Phase | Version | Content | Exit criteria |
|---|---|---|---|
| v0.1 | 0.1.0 | F1–F7 code search MVP | US-101..105, US-201..205, US-301..303 ACs green; dogfooded on KanPad for 2 weeks |
| v0.2 | 0.2.0 | F8–F10 eval harness | Eval report committed; baseline vs tuned numbers; US-401..404 ACs green |
| v0.3 | 0.3.0 | F11 git intelligence | `git_search` usable; US-501 ACs green |
| v0.4 | 0.4.0 | F12 OSS release | CI green; release binaries; docs live; S6 |
