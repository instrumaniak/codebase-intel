# ADR 002 — SQLite as the single store (FTS5 + sqlite-vec), no external vector DB

## Status: Accepted

## Date: 2026-11

## Context

Retrieval needs: (1) BM25 keyword/substring search over chunk text including code identifiers, (2) approximate nearest-neighbor search over ~5k float32[768] vectors (growing to maybe 50k), (3) transactional manifest + metadata, (4) concurrent reader access during reindexing. Alternatives evaluated: Qdrant, LanceDB, PostgreSQL+pgvector, FAISS, pure in-Rust cosine over a BLOB column, SQLite alone.

## Decision

One SQLite file per project: FTS5 (trigram) for BM25, the `sqlite-vec` extension (`vec0`-style, bundled via the `sqlite-vec` crate) for vector search, WAL mode, mmap enabled.

## Reasons

### One artifact, zero services for everything
- Qdrant/LanceDB both ship the vector half but leave you to bolt on FTS and the manifest somewhere else — two stores, two backup stories, two failure modes. SQLite does all three.
- No daemon (ADR 001) is only credible if the store is embedded. Postgres/pgvector fails that constraint outright; Qdrant can run embedded but still splits keyword vs vector logic across systems.
- FAISS solves ANN at 100x our scale and adds nothing at 5k vectors; exact cosine over 5k×768 floats is ~15 MB of reads — trivial.

### Extension risk is contained
- `sqlite-vec` ships a bundled build via crate features; if the extension ever fails to load, architecture.md §4.2 keeps a documented fallback: BLOB chunks + cosine in Rust over candidate ids from FTS. The failure mode is "slightly slower vector search", never "tool broken".

### FTS5 trigram fits identifiers
- Identifiers like `handleJwtRefresh` tokenize fine under trigram (substring match `JwtRefr`), which Unicode61 would shred less usefully for code. BM25 gives the exact-match tier of the hybrid for free.

### Operational floor is unbeatable for a personal-project/OSS tool
- Backup = copy one file. Debug = `sqlite3 index.db .tables`. Corrupt = rebuild (≤ 1 min). No Docker, no ports, no permissions.

## Consequences

- No server-side HNSW at 10x scale; we accept brute-force exact cosine (it is *exact* — a correctness property, not a bug) up to "the day we need > 100k chunks".
- Schema migrations mean drop-and-rebuild until post-1.0; acceptable while schema is young (architecture.md §10).
- Write concurrency is single-writer by design; per-file transactions bound contention (≤ 200 ms).

## Revisit Trigger

- Index exceeds ~150k chunks or query latency regresses; evaluate LanceDB (embedded, Rust-native) before Qdrant.
- Multi-user/team scenarios (v2+) requiring concurrent writes — reassess Postgres + pgvector.
