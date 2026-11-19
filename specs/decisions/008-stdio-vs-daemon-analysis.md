# ADR 008 — Stdio over daemon (and Postgres/Redis/Docker) for the single-user case

## Status: Accepted

## Date: 2026-11

## Context

ADR 001 picked stdio MCP (spawned per client session) and deferred the question of a long-lived background service. The question came up concretely: an agent continuously searching/reading/writing files — would a lightweight always-on service be better, possibly backed by Postgres/pgvector or Redis, possibly in Docker? This ADR records the analysis so it is not relitigated from scratch.

The workload being designed for:

- One developer, typically one MCP client session (occasionally two windows)
- Tens to low-hundreds of searches per session — human-paced, not high-QPS
- Index scale: ~5k chunks now, 50k ceiling (PRD scope); ≤ ~150 MB DB (NFR-10)
- Warm query ≈ 100–150 ms, dominated by the Ollama query-embedding call (~10–30 ms for SQL + fusion)
- Index content changes only when the working tree changes — observable via content hashes, so no push notification is required for correctness

## Decision

**Stdio stays canonical for v0.1–v0.3.** No daemon, no Postgres, no Redis, no Docker requirement. If (and only if) dogfooding produces evidence that staleness or multi-window spawn contention hurts, add an **optional** `serve --http` / Unix-socket mode (same binary, same code path), optionally installed as a systemd user unit — still SQLite-backed. Postgres/Redis/Docker are not on the roadmap for the single-user tool.

## Reasons

### The compute is already daemonized — it's Ollama
- The latency-dominant step (query embedding) already runs in a resident service (Ollama, with model warm-keep). A second daemon adds coordination around the cheap 10% (SQL + fusion ≈ 10–30 ms) and cannot beat mmap'd SQLite by more than that minus spawn cost — which stdio amortizes to ≈ 0 per call within a session (spawn happens once per session, not per query).

### mmap already shares hot pages across processes
- Two opencode windows each spawn a server; both read the same SQLite file with `mmap_size=256 MB`. The OS page cache serves hot pages (vectors, FTS index) from shared memory — the main warm-state advantage of a daemon largely disappears without any extra process.

### Per-file transactions + WAL make writers invisible to readers
- Reindex writes ≤ 200 ms txns; agents never block on the indexer, and the indexer never blocks agents (ADR 007). This neutralizes the "continuous file churn during dev/tests" concern specifically for this tool — it never writes repo files and is off the editor/test-runner critical path by construction.

### Postgres + pgvector solves a problem we don't have
- Target benefit would be approximate search (HNSW) at 100k+ chunks. At 5–50k, exact cosine over mmap'd data is milliseconds and *exact*. Postgres costs: container lifecycle, connection management, migration discipline, backup procedure (SQLite backup = one file copy), and a hard Docker dependency in an open-source install path whose target users may run on constrained/shared hosts (the project's own precedent: cPanel deployment experience).

### Redis solves a problem we don't have either
- A cache pays off on repeated keys with high hit rates. Code-search queries are rarely identical in a session; when they are, the index answers in ~30 ms anyway. Worst case, a stale cache entry after reindex is worse than a recompute — cache invalidation enters the bug budget for zero measured gain.

### The watcher is the only real daemon upside, and ADR 007 covers it with less machinery
- Live inotify/FSEvents reindex inside a daemon is the one genuine upgrade. It is also the riskiest: 10k-event bursts from `cargo clean`, rustc's write-via-rename pattern defeating naive watchers, debounce/coalesce complexity — all to deliver freshness that a pull-based manifest check provides on demand for ~50 ms per query, plus ≤ 1 s no-change reindex. Not worth it in v1; opt-in later if wanted.

### Operational surface is the product for OSS
- "Clone, `ollama pull`, one config line, works" beats "…and enable a service / run Docker" every time. Every additional moving part taxes install conversion, issue-report reproduction, and documentation.

## Consequences

- Two concurrent clients both query fine; only reindex serializes (`<db>.lock`, non-blocking, loser reports `busy`).
- Agents may observe a stale index between edits; mitigated by staleness notes (US-104) and the `index_status`/`reindex` tools; instruction text tells agents when to refresh.
- If evidence later justifies a daemon, it arrives as `serve --http <socket>` reusing the same `retrieve`/`store` modules — not as new infrastructure, not as a change to the storage layer, not with Redis/Postgres attached.
- Docker may appear later as an *optional* packaging for hosted/team offerings (P4 persona, PRD §2), with its own ADR — explicitly not for the single-user tool.

## Revisit Trigger

- Dogfood metrics (2-week window on KanPad): > 2 stale-index warnings per session reported by agents, or repeated `busy` reindex failures across windows, or agents routinely missing obviously-edited files.
- A concrete external requirement (sandboxed client without spawn ability) arrives.
- Index scale exceeds ~150k chunks or NFR-1 misses after tuning — then evaluate LanceDB (embedded, no daemon) *before* any server/postgres option (ADR 002 trigger).
- Team/multi-user usage becomes a stated goal — separate ADR, separate product decision.
