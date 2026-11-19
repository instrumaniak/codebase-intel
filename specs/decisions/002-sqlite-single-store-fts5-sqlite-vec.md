# ADR 002 — SQLite as the single store (FTS5 + sqlite-vec), no external vector DB

## Status: Accepted

**Revision note (2026-11-19, pre-implementation spec review):** vector store pinned to plain-table + scalar `vec_distance_cosine` (the design architecture.md §4.2/§5 actually specifies) instead of "vec0-style"; the FTS-candidate-limited Rust fallback was removed; raw float32 BLOBs confirmed valid direct inputs to the distance functions.

## Date: 2026-11

## Context

Retrieval needs: (1) BM25 keyword/substring search over chunk text including code identifiers, (2) exact nearest-neighbor search over ~5k float32[768] vectors (growing to maybe 50k), (3) transactional manifest + metadata, (4) concurrent reader access during reindexing. Alternatives evaluated: Qdrant, LanceDB, PostgreSQL+pgvector, FAISS, pure in-Rust cosine over a BLOB column, SQLite alone.

## Decision

One SQLite file per project: FTS5 (trigram) for BM25; vectors as **raw little-endian float32 BLOBs in a plain `vec_chunks` table**, ranked exactly with sqlite-vec's scalar `vec_distance_cosine` function (no vec0 / virtual table); WAL mode, mmap enabled. If the extension fails to load, the identical query runs in Rust over the same filtered row set.

## Reasons

### One artifact, zero services for everything
- Qdrant/LanceDB both ship the vector half but leave you to bolt on FTS and the manifest somewhere else — two stores, two backup stories, two failure modes. SQLite does all three.
- No daemon (ADR 001) is only credible if the store is embedded. Postgres/pgvector fails that constraint outright; Qdrant can run embedded but still splits keyword vs vector logic across systems.
- FAISS solves ANN at 100x our scale and adds nothing at 5k vectors; exact cosine over 5k×768 floats is ~15 MB of reads — trivial.

### Plain table + scalar functions beat vec0 for our queries
- sqlite-vec's scalar functions (`vec_distance_cosine`, `vec_distance_L2`) operate on any vector column — no vec0 virtual table required. Vectors are written as plain BLOBs via rusqlite (raw LE float32, length divisible by 4 — valid direct input to the distance functions; no `vec_f32()` subtype ceremony).
- kNN over the plain table is a brute-force O(n) scan — at 5k×768 floats ≈ 15 MB of reads, single-digit ms. It is *exact*, and exactness is a correctness property we already accept (see Consequences). vec0 provides no ANN either at this scale.
- The decisive reason: pre-fusion filters (US-203 — path_prefix / kind / lang) compose with the vector scan as an ordinary SQL `WHERE` over `JOIN chunks` — vec0's kNN `MATCH` cannot filter by other columns without post-filtering, which would break US-203 AC4 ("filtered searches return real top-N within the filter").
- Extension risk is contained: if `sqlite-vec` fails to load, the **same** query executes in Rust over the **same** filtered row set — slower scan, identical rows and results, never "tool broken". (The earlier "cosine in Rust over candidate ids from FTS" was a third, worse design: degraded results must not depend on which candidate list won. It was removed.)

### FTS5 trigram fits identifiers
- Identifiers like `handleJwtRefresh` tokenize fine under trigram (substring match `JwtRefr`), which Unicode61 would shred less usefully for code. BM25 gives the exact-match tier of the hybrid for free.

### Operational floor is unbeatable for a personal-project/OSS tool
- Backup = copy one file. Debug = `sqlite3 index.db .tables`. Corrupt = rebuild (≤ 1 min). No Docker, no ports, no permissions.

## Consequences

- Brute-force exact cosine is the design (no HNSW at this scale — it is *exact*, a correctness property, not a bug) up to "the day we need > 100k chunks" (revisit trigger).
- Schema migrations mean drop-and-rebuild until post-1.0; acceptable while schema is young (architecture.md §10).
- Write concurrency is single-writer by design; per-file transactions bound contention (≤ 200 ms).

## Revisit Trigger

- Index exceeds ~150k chunks or query latency regresses; evaluate LanceDB (embedded, Rust-native) before Qdrant.
- Multi-user/team scenarios (v2+) requiring concurrent writes — reassess Postgres + pgvector.
