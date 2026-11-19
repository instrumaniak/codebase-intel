# Architecture — codebase-intel

**Status:** v1 draft — awaiting owner review
**Companion docs:** [prd.md](prd.md) · [eval-spec.md](eval-spec.md) · [epics.md](epics.md) · ADRs in [../decisions/](../decisions/)

---

## 1. Context and constraints

`codebase-intel` runs as a **stdio MCP server**, spawned on demand by the MCP client (opencode first). It has no daemon, no port, no account, and no runtime dependency other than the OS and — optionally — a localhost Ollama for query embeddings. It must be invisible while development and tests run beside it.

Hard constraints (from PRD NFR-1..13):

- Warm query ≤ 150 ms p50 (excluding embedder roundtrip, ≤ 600 ms with it)
- Cold start ≤ 500 ms; idle CPU 0%; RSS ≤ 100 MB on a 1k-file index
- Safe under concurrent `cargo test`/`npm run dev` storms; never corrupts the repo
- 100% local; network = localhost Ollama only; no telemetry
- Linux first-class, macOS supported, Windows unsupported (mmap + `O_EXLOCK` specifics)

These constraints — not preference — drive every decision below (see also [ADR 008](../decisions/008-stdio-vs-daemon-analysis.md) for the stdio-vs-daemon-vs-Postgres/Redis/Docker analysis).

## 2. System overview

```
                ┌──────────────────── client (opencode, Claude Code, …) ───────────────────┐
                │  spawns: codebase-intel (stdio)  ── JSON-RPC MCP ──▶ tool calls          │
                └──────────────────────────────────────────────────────────────────────────┘
                                                │
┌────────────────────────────── codebase-intel ──────────────────────────────────────────┐
│                                                                                        │
│  serve ──▶ mcp::Server (rmcp)                                                          │
│              tools: code_search · get_file_outline · index_status · reindex            │
│              instructions injected at handshake                                         │
│                                                                                        │
│  index/reindex ─▶ Indexer ───────────────┐                                             │
│    walker (ignore) ─▶ chunker (tree-sitter +           ┌──────────────┐                │
│    line-window fallback + gap fill) ─▶    │            │  SQLite DB   │                │
│    content hash (blake3) ─▶ embedder ────►│            │  (WAL, mmap) │                │
│    (Ollama /api/embed, batch)             │            └───┬───────┬──┘                │
│                                            │   writes:      │       │ reads:           │
│                                            ▼                ▼       ▼                  │
│                                     files · chunks · chunks_fts · vec_chunks · meta    │
│                                                                                        │
│  code_search ◀── Retriever:                                                            │
│    query ─▶ embed (Ollama) ─▶ vector top-32                                             │
│           └─▶ FTS5 (trigram, BM25) top-32 ─▶ filters (SQL) ─▶ RRF (k=60)                │
│              ─▶ symbol boost ─▶ staleness note ─▶ compact JSON                          │
│                                                                                        │
│  mine-sessions (v0.2) ─▶ offline snapshot miner ─▶ eval.jsonl                          │
│  eval           (v0.2) ─▶ metrics: Recall@k, MRR, nDCG ─▶ tuning report                │
│  git index      (v0.3) ─▶ gix walk: subjects/bodies embed + churn table                │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

## 3. Repository layout and modules

Single crate, one binary (`codebase-intel`), library + thin main. Single crate is deliberate — no publishable-crate granularity needed yet.

```
src/
  main.rs            # clap entry, logging init, top-level error mapping
  cli.rs             # subcommands: serve (default) · index · status · mine-sessions (v0.2) · eval (v0.2)
  config.rs          # index location, Ollama URL, model id, limits — env vars + XDG paths
  walker.rs          # gitignore-aware walk, size/type filters, secret exclusion list
  chunker/
    mod.rs           # Chunk { path, start, end, kind, symbol, parent, doc, text, lang, hash }
    treesitter.rs    # grammar dispatch, query sets per language
    fallback.rs      # line-window chunker (unsupported langs, oversized symbols)
    gaps.rs          # coverage invariant: every byte in ≤1 chunk
  embedder.rs        # Embedder trait + OllamaEmbedder (reqwest, batch=32, timeout, circuit breaker)
                     #   + NullEmbedder (FTS-degraded mode, explicit)
  store/
    mod.rs           # Store handle: open, migrate, WAL+mmap pragmas, busy_timeout
    schema.sql       # DDL (embedded, versioned via meta.schema_version)
    files.rs         # manifest: path, blake3, mtime, size
    chunks.rs        # chunk CRUD, delete-by-file (txn)
    fts.rs           # chunks_fts (trigram) maintenance + BM25 query
    vec.rs           # vec_chunks (sqlite-vec) ops
  retrieve/
    mod.rs           # hybrid pipeline: embed?, fts, fuse (RRF), boosts, filters, staleness
    rrf.rs           # reciprocal rank fusion — pure, deterministic, unit-tested
    boost.rs         # symbol-name boost rules (exact/prefix)
  mcp/
    mod.rs           # rmcp server, tool registration, handshake instructions
    tools.rs         # code_search · get_file_outline · index_status · reindex
  gitmeta.rs         # current HEAD, short rev, commit count — via gix
  util.rs            # canonical repo root, project hash, XDG paths, nice(19)
tests/
  fixtures/          # synthetic fixture repo (tiny, deterministic) — never a real private repo
  integration_*.rs   # end-to-end over fixture repo with mocked Ollama (wiremock)
  retrieval_quality.rs  # golden queries → expected files (gate for ranking changes)
```

## 4. The index database

### 4.1 Location

`$XDG_DATA_HOME/codebase-intel/indexes/<project-slug>-<blake3(canonical_root)[..12]>.db`
(defaults: `~/.local/share/codebase-intel/indexes/…`; override `CODEBASE_INTEL_HOME`; one DB per project root, keyed by canonicalized absolute path hash).

A sibling lock file `<db>.lock` (`O_EXLOCK`) serializes concurrent reindex attempts (PRD US-302/AC4).

### 4.2 Schema (v1)

```sql
PRAGMA journal_mode = WAL;
PRAGMA synchronous  = NORMAL;   -- safe with WAL; we never need FULL durability for a rebuildable index
PRAGMA mmap_size    = 268435456; -- 256 MB mmap ceiling for read pages
PRAGMA busy_timeout = 500;      -- reindexing should not starve queries

CREATE TABLE meta (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL
);
-- keys: schema_version, project_root, indexed_at, git_head, embed_model, embed_dims,
--       index_run_id, full_index_completed (bool), ollama_reachable (last probe)

CREATE TABLE files (
  path          TEXT PRIMARY KEY,        -- repo-relative, '/' separators
  content_hash  TEXT NOT NULL,           -- blake3 hex
  mtime_ms      INTEGER NOT NULL,
  size_bytes    INTEGER NOT NULL,
  indexed_at    INTEGER NOT NULL
);

CREATE TABLE chunks (
  id         INTEGER PRIMARY KEY,
  file_path  TEXT NOT NULL REFERENCES files(path) ON DELETE CASCADE,
  lang       TEXT NOT NULL,
  kind       TEXT NOT NULL,              -- function|method|class|struct|impl|trait|enum|module|doc|gap|other
  symbol     TEXT,                       -- enclosing/defined symbol, nullable (gap chunks)
  parent     TEXT,                       -- breadcrumb of enclosing symbols
  start_line INTEGER NOT NULL,
  end_line   INTEGER NOT NULL,
  start_byte INTEGER NOT NULL,
  end_byte   INTEGER NOT NULL,
  text       TEXT NOT NULL,              -- full chunk text (header included)
  text_hash  TEXT NOT NULL               -- for embed dedup across files
);
CREATE INDEX chunks_file_idx ON chunks(file_path);
CREATE INDEX chunks_symbol_idx ON chunks(symbol);

CREATE VIRTUAL TABLE chunks_fts USING fts5(
  text, path, symbol,
  content='chunks', content_rowid='id',
  tokenize='trigram'
);

CREATE TABLE vec_chunks (
  chunk_id INTEGER PRIMARY KEY REFERENCES chunks(id) ON DELETE CASCADE,
  embedding BLOB NOT NULL                -- float32[768] little-endian
);
-- cosine distance computed by sqlite-vec (vec_distance_cosine) or in-Rust if extension unavailable

CREATE TABLE index_runs (                -- audit trail; used by status + eval
  run_id INTEGER PRIMARY KEY,
  started_at INTEGER NOT NULL,
  finished_at INTEGER,
  mode TEXT NOT NULL,                    -- full|incremental
  files_changed INTEGER NOT NULL DEFAULT 0,
  chunks_written INTEGER NOT NULL DEFAULT 0,
  ok INTEGER NOT NULL DEFAULT 0
);
```

Row estimates at KanPad scale (723 files, ~190 tsx / 161 md / 118 json / 105 rs / 85 ts): ~4–6k chunks; vectors 768 float32 = 3 KB each ⇒ ~18 MB; FTS trigram ≈ 2–3× source bytes (~10–20 MB); total DB ≤ 150 MB (NFR-10).

### 4.3 Coverage invariant (AST chunks + gap spans)

Chunker emits **symbol chunks** (body + attached doc) and fills everything else with **gap chunks** (line windows ≤ 120 lines, split at blank lines if possible, overlapping the previous window by 4 lines). Every byte of every indexed file is in exactly one chunk. Non-parseable/oversized files → pure line-window chunks with `kind='gap'`, `symbol=NULL`. This is what makes "finds relevant content without missing files" mechanically true — embeddings can't see bytes that aren't chunks.

### 4.4 Concurrency model

- **Readers**: queries run on the calling thread against a WAL snapshot. They never block writers and writers never block readers. A query that arrives mid-reindex sees the pre-reindex state — acceptable and explicitly rate-limited by staleness note (US-104).
- **One writer process**: reindex/index takes `<db>.lock` (`O_EXLOCK`, non-blocking). A concurrent server that loses the lock reports `reindex: busy` and serves stale results with a note.
- **Small write transactions**: per-file batch (delete rows for path → insert chunks → FTS/vec rows → commit) targets ≤ 200 ms txns; readers never see a half-written file's rows because delete+insert are atomic per file.
- **Reduced CPU priority**: the indexing thread calls `nice(19)` (Unix); design rule: the query path and the MCP server event loop contain no indexing logic — indexing work enters only via explicit `reindex` call or an opt-in `serve --index-on-start=if-stale` (off by default in v1).
- **Memory**: prepared statements reused per connection; vector scan materializes only top-k via sqlite-vec; no full-table loads into memory; RSS ≤ 100 MB (NFR-4).

## 5. Retrieval pipeline

```
query (string, optional filters)
  │
  ├─ if Ollama reachable (probe cached ≤30s): embed(query) ──► vec candidates:
  │      SELECT chunk_id, distance FROM vec_chunks ORDER BY vec_distance_cosine(embedding, ?) LIMIT 32
  │
  └─ FTS candidates (always):
        SELECT rowid, bm25(chunks_fts) FROM chunks_fts
        WHERE chunks_fts MATCH <quoted-escaped query> ORDER BY bm25(chunks_fts) LIMIT 32
  │
  ├─ filters (SQL, pre-fusion): path_prefix / kind / lang → candidate id allow-set
  │
  ├─ fusion: RRF k=60 over the two ranked lists
  │     score(d) = Σ_lists 1 / (k + rank_l(d))   · sources weighted 1.0 (equal) in v1
  │
  ├─ boosts (post-fusion, additive tiny weights — never dominate):
  │     symbol exact  +0.50 · symbol prefix +0.25 · path component match  +0.10
  │     recency (v0.3 via git churn)              ≤ +0.05
  │
  ├─ exact-identifier short-circuit: if query matches ^[A-Za-z_][A-Za-z0-9_]{2,}$
  │     AND FTS returns ≥ 3 hits ⇒ FTS-only ranking for those, vector appended after (tagged)
  │
  └─ shape: top `limit` (10) chunks → compact JSON (§6) with repo-relative paths,
       line spans, symbol/kind, fused score, snippet (≤ 20 lines, ≤ 2,000 tokens total),
       plus one-line staleness note when applicable
```

Determinism (NFR-11): RRF and boosts are pure functions of ranks/text; tie-break by `(score, chunk_id)`; no clocks in the ranking path.

Degraded mode (US-105): Ollama down ⇒ vec list empty ⇒ RRF over FTS only; results tagged `degraded: true`; note appended; recovery probed lazily next call (≤ 30 s cache).

## 6. MCP surface

### 6.1 Transport and negotiation

stdio, NDJSON JSON-RPC, `rmcp` crate. Server name `codebase-intel`, instructions (emitted in `initialize`):

> Prefer `code_search` before `grep` for conceptual queries ("where is X handled", "what does Y do"). Follow with `read` for exact code. Use `grep` for exact strings. If results look stale, call `index_status`; run `reindex` only when needed.

### 6.2 Tool schemas (v0.1)

```jsonc
// code_search
{ "query": string, "limit"?: int (1..25, default 10),
  "path_prefix"?: string, "kind"?: string, "lang"?: string }
→ { "results": [ { "path": "src/auth/refresh.rs", "start_line": 42, "end_line": 71,
                   "symbol": "refresh_token", "kind": "function",
                   "score": 0.0321, "snippet": "..." } ],
    "degraded": false, "index_age_secs": 3612, "stale": false }

// get_file_outline
{ "path": "src/auth/refresh.rs" }
→ { "symbols": [ { "kind": "function", "name": "refresh_token", "parent": null,
                   "start_line": 40, "end_line": 71, "doc": "Refresh an expiring …" } ] }

// index_status
{} → { "schema_version": 1, "project_root": "…", "indexed_files": 711, "chunks": 5210,
       "git_head_indexed": "…", "git_head_current": "…", "stale": false,
       "embedder": { "model": "nomic-embed-text", "dims": 768, "reachable": true },
       "last_run": { "mode": "incremental", "finished_at": "…", "ok": true } }

// reindex
{ "mode"?: "incremental" | "full" }     // default incremental
→ { "mode": "incremental", "files_changed": 3, "chunks_written": 41,
    "duration_ms": 812, "ok": true }    // busy → { "ok": false, "reason": "busy" }, HTTP-free
```

Error model: tool-level `isError` with short actionable message; never raw SQL/panics.

## 7. Failure modes

| Failure | Behavior | Recovery |
|---|---|---|
| Ollama down/timeout | FTS-only results, `degraded: true` | automatic re-probe |
| Ollama wrong model dims | refuse vector ops; log; FTS-only until `reindex --full` re-embeds | `reindex --full` after pulling correct model |
| Index DB corrupt | open error → `index_status` reports `corrupt`; search refuses | `index --full` (deletes + rebuilds; safe) |
| Kill -9 mid-reindex | per-file txns ⇒ no partial file rows; manifest/index_runs truncate at last commit | next run reconciles by hash; `--full` if paranoid |
| Two servers reindex concurrently | loser: `busy`, serves stale + note | n/a |
| Disk full during index | txn rollback; `ok:false` with reason; old chunks for changed file intact (delete+insert atomic) | free space, re-run |
| File vanishes between walk & read | file skipped, counted in report, run continues | n/a |
| Fixture/test DB misuse | tests use tempdir DBs only; production path never touched in tests | enforced by `testutil` |

## 8. Security and privacy

- **Project-root confinement**: walker starts at resolved project root; canonicalization rejects symlink escapes (a symlink pointing outside the root is skipped). `get_file_outline` / snippets never emit content from outside the root.
- **Secret exclusion** (walker): `.env*`, `*.pem`, `*.key`, `*.p12`, `*.pfx`, `*.keystore`, `*.jks`, `id_rsa*`, `id_ed25519*`, `id_dsa*`, `credentials*.json`, `secrets*.{yaml,yml,json,toml}`, `.npmrc`, `.pypirc`, `*.token`, `.envrc`. Respects `.gitignore`/`.git/info/exclude` first (already-tracked secrets are the user's problem, not ours to silently index — still excluded by name).
- **Network**: only `http://localhost:11434` (configurable via `CODEBASE_INTEL_OLLAMA_URL` to another *local or trusted* address; documented as such — code content would be sent there). No DNS, no telemetry, no update checks (NFR-8). CI grep-gate: no `reqwest` calls outside `embedder.rs` + `gitfetch.rs` (v0.3, none planned).
- **Telemetry**: none. Ever. `index_runs` stays local.
- **Phase-2 privacy**: session snapshots are copied with SQLite's `.backup` into a temp dir, mined offline, then deleted by default; mined JSONL for *private* use stays out of git (`.gitignore`d); public fixtures come from public repos only (eval-spec.md §7).
- **MCP trust boundary**: stdio local process inherits user privileges — like git. Server never writes outside `indexing` code path, and repo files are read-only to us.

## 9. Testing strategy

| Layer | What | Tooling |
|---|---|---|
| Unit | RRF fusion math, symbol boost ranking, gap-span coverage invariant (every byte covered, no overlap), trigram escaping, canonicalization/symlink logic, config parsing | built-in `cargo test` |
| Property-ish | Random file corpus → chunk → reassemble == original bytes (coverage invariant); 10k random trees | hand-rolled with seeded RNG |
| Integration | index fixture repo (mocked Ollama via `wiremock`) → search → assert expected files in top-5; degraded mode; reindex x2 idempotence; kill-mid-index → status → full rebuild | tempdir + wiremock |
| Retrieval quality | golden queries vs fixture repo; Recall@k/MRR floors; **fails CI on regression** (after v0.2 defines fixtures) | `retrieval_quality.rs` |
| CLI contract | `index`, `status`, `reindex` output shapes parse as JSON; exit codes | assert_cmd style |
| Perf smoke | index build < 60 s; no-change incremental < 1 s; query < 150 ms on fixture | feature-gated `bench` tests, run manually/weekly |

Coverage floor: **80% lines** (`cargo llvm-cov`), enforced at release; unit+integration combined. Ollama-dependent tests are feature-gated (`--features live-ollama`) and excluded from default runs (NFR-9).

## 10. Build, release, compatibility

- Rust edition 2024; **MSRV = current stable − 2** (at time of writing: 1.88); toolchain pinned by `rust-toolchain.toml`
- Deps (first cut): `rmcp`, `rusqlite` (bundled), `sqlite-vec` (bundled, runtime `load_extension`), `tree-sitter` + grammars (rust, typescript, javascript, markdown, sql, yaml, toml, json), `ignore`, `blake3`, `reqwest` (rustls), `tokio`, `clap`, `gix`, `serde`/`serde_json`, `tracing`/`tracing-subscriber`, `thiserror`, `anyhow` (main only)
- Release: `cargo build --release` with `strip = true`, `lto = "thin"`, single codegen unit; `cargo-dist` for cross-builds (x86_64-unknown-linux-gnu, aarch64-…, macOS arm64/x86_64); crates.io publish as `codebase-intel`
- DB schema migrations: none until v0.2+ if unavoidable; until then, `schema_version` bump ⇒ drop-and-rebuild (documented, rebuild is ≤ 1 min)
- Linux: primary (dev machine, CI). macOS: CI compile+test. Windows: unsupported; code avoids POSIX-only APIs behind `#[cfg(unix)]` but no support promise

## 11. Module dependency rules

- `store` depends on nothing project-internal except `util`
- `retrieve` depends on `store`, `embedder`
- `mcp` depends on `retrieve`, `store`, `gitmeta`
- `indexer` (walker/chunker) depends on `store`, `embedder`, `util`
- **No cycles**; enforced by `cargo` module layering + a simple `deps` test listing allowed pairs
- `eval` (v0.2) is a separate binary target sharing `store`/`retrieve` read-only — it must never write to the index
