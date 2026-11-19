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
- Linux, macOS and Windows are all first-class targets (ADR 009); platform-specific behavior is confined to `util.rs` + the walker, and the file lock is one std code path on all three

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
  main.rs            # clap entry, logging init (stderr only — stdout is the MCP stream), top-level error mapping
  cli.rs             # subcommands: serve (default) · index · status · mine-sessions (v0.2) · eval (v0.2)
                     #   status = one-shot index_status (JSON to stdout); the MCP `reindex` tool is the
                     #   same code path as `index` (incremental) — one stats struct, one <db>.lock
  config.rs          # index location, Ollama URL, model id, limits — env vars + platform data dir (via `directories`)
  walker.rs          # gitignore-aware walk, size/type filters, secret exclusion list
  chunker/
    mod.rs           # Chunk { path, start, end, kind, symbol, parent, doc, text, lang, hash }
    treesitter.rs    # grammar dispatch, query sets per language
    fallback.rs      # line-window chunker (unsupported langs, oversized symbols)
    gaps.rs          # coverage invariant: every byte in exactly one chunk (partition, not just cover)
  embedder.rs        # Embedder trait + OllamaEmbedder (reqwest, batch=32, timeout, circuit breaker)
                     #   + NullEmbedder (FTS-degraded mode, explicit)
  store/
    mod.rs           # Store handle: open, migrate, WAL+mmap+foreign_keys pragmas, busy_timeout, PRAGMA optimize (end of index runs)
    schema.sql       # DDL (embedded, versioned via meta.schema_version) — includes FTS sync triggers + kind/mode CHECKs
    files.rs         # manifest: path, blake3, mtime, size
    chunks.rs        # chunk CRUD, delete-by-file (txn)
    fts.rs           # chunks_fts (trigram) queries + BM25 — row maintenance is trigger-enforced, never app code
    vec.rs           # vec_chunks ops — exact cosine via sqlite-vec scalar fns on a plain table (no vec0)
  retrieve/
    mod.rs           # hybrid pipeline: embed?, fts, fuse (RRF), boosts, filters, staleness
    rrf.rs           # reciprocal rank fusion — pure, deterministic, unit-tested
    boost.rs         # symbol-name boost rules (exact/prefix)
  mcp/
    mod.rs           # rmcp server, tool registration, handshake instructions
    tools.rs         # code_search · get_file_outline · index_status · reindex
  gitmeta.rs         # current HEAD, short rev, commit count — via gix
  util.rs            # canonical repo root, project hash, platform data dir (BaseDirectories), background-class priority setter
                     #   ONLY module allowed platform cfg (#[cfg(unix)] / #[cfg(windows)]) — enforced by the `deps` test (§11)
tests/
  fixtures/          # synthetic fixture repo (tiny, deterministic) — never a real private repo
  integration_*.rs   # end-to-end over fixture repo with mocked Ollama (wiremock)
  retrieval_quality.rs  # golden queries → expected files (gate for ranking changes)
```

## 4. The index database

### 4.1 Location

`<platform-data-dir>/codebase-intel/indexes/<project-slug>-<blake3(canonical_root)[..12]>.db`
where the data dir is resolved by `directories::BaseDirectories::data_dir()`:

| Platform | Index location |
|---|---|
| Linux | `$XDG_DATA_HOME/codebase-intel/indexes/…` (default `~/.local/share/codebase-intel/indexes/…`) |
| macOS | `~/Library/Application Support/codebase-intel/indexes/…` |
| Windows | `%APPDATA%\codebase-intel\indexes\…` |

Override: `CODEBASE_INTEL_HOME` (unchanged). One DB per project root, keyed by canonicalized absolute path hash — the hash input is the platform-canonical path (e.g. `\\?`-prefixed on Windows), consistent per machine; indexes are per-machine artifacts by design.
**The index must live on a local disk**: WAL does not work reliably on network filesystems, and neither does mmap — the default location is local on every platform, and `CODEBASE_INTEL_HOME` pointed at a network drive is unsupported and documented as such.

A sibling lock file `<db>.lock` (std `File::try_lock` — `flock LOCK_EX|LOCK_NB` on Unix, `LockFileEx … FAIL_IMMEDIATELY` on Windows — one code path) serializes concurrent reindex attempts (PRD US-302/AC4). Advisory, released on process exit (handle close); the lock file itself may persist — it is a sentinel, not state. On Windows the file is opened with read+write (not append-only) — `try_lock` fails on append-only handles.

### 4.2 Schema (v1)

```sql
PRAGMA journal_mode = WAL;
PRAGMA synchronous  = NORMAL;   -- safe with WAL; we never need FULL durability for a rebuildable index
PRAGMA mmap_size    = 268435456; -- 256 MB mmap ceiling for read pages
PRAGMA busy_timeout = 500;      -- reindexing should not starve queries
PRAGMA foreign_keys = ON;       -- PER CONNECTION (Store::open applies it); ON DELETE CASCADE is inert without it

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
  kind       TEXT NOT NULL CHECK (kind IN ('function','method','class','struct','impl','trait','enum','module','doc','gap','other')),  -- canonical kind set; PRD US-203 filters against this
  symbol     TEXT,                       -- enclosing/defined symbol, nullable (gap chunks)
  parent     TEXT,                       -- breadcrumb of enclosing symbols
  start_line INTEGER NOT NULL,
  end_line   INTEGER NOT NULL,
  start_byte INTEGER NOT NULL,
  end_byte   INTEGER NOT NULL,
  text       TEXT NOT NULL,              -- full chunk text (header included)
  text_hash  TEXT NOT NULL               -- for embed dedup; store embeds each distinct text_hash once per run, reuses the vector for identical chunks across files
);
CREATE INDEX chunks_file_idx ON chunks(file_path);
CREATE INDEX chunks_symbol_idx ON chunks(symbol);

CREATE VIRTUAL TABLE chunks_fts USING fts5(
  text, file_path, symbol,   -- FTS column names MUST match the content table's columns (FTS5 external-content rule)
  content='chunks', content_rowid='id',
  tokenize='trigram'
);
-- External-content FTS5 tables never sync themselves. Consistency is trigger-enforced inside the
-- same per-file transaction — store/fts.rs contains QUERIES and integrity-check only, never row maintenance.
CREATE TRIGGER chunks_ai AFTER INSERT ON chunks BEGIN
  INSERT INTO chunks_fts(rowid, text, file_path, symbol) VALUES (new.id, new.text, new.file_path, new.symbol);
END;
CREATE TRIGGER chunks_ad AFTER DELETE ON chunks BEGIN
  INSERT INTO chunks_fts(chunks_fts, rowid, text, file_path, symbol)
  VALUES ('delete', old.id, old.text, old.file_path, old.symbol);
END;
CREATE TRIGGER chunks_au AFTER UPDATE ON chunks BEGIN
  INSERT INTO chunks_fts(chunks_fts, rowid, text, file_path, symbol)
  VALUES ('delete', old.id, old.text, old.file_path, old.symbol);
  INSERT INTO chunks_fts(rowid, text, file_path, symbol) VALUES (new.id, new.text, new.file_path, new.symbol);
END;

CREATE TABLE vec_chunks (
  chunk_id INTEGER PRIMARY KEY REFERENCES chunks(id) ON DELETE CASCADE,
  embedding BLOB NOT NULL                -- raw float32[768] little-endian (3072 bytes; vec_f32() not required)
);
-- Exact brute-force cosine via sqlite-vec's scalar functions on a REGULAR column:
--   SELECT c.id, vec_distance_cosine(v.embedding, ?) AS d
--   FROM vec_chunks v JOIN chunks c ON c.id = v.chunk_id
--   WHERE <filters — pre-fusion>  ORDER BY d LIMIT 32
-- Raw LE float32 BLOBs (length divisible by 4) are valid direct inputs to vec_distance_cosine.
-- A plain table (not vec0) is deliberate: vec0 kNN cannot filter by other columns without
-- post-filtering, which would break "real top-N within the filter" (US-203 AC4). See ADR 002.
-- If the extension fails to load: the identical query runs in Rust over the same filtered row
-- set — slower scan, same row set, never broken.

CREATE TABLE index_runs (                -- audit trail; used by status + eval
  run_id INTEGER PRIMARY KEY,
  started_at INTEGER NOT NULL,
  finished_at INTEGER,
  mode TEXT NOT NULL CHECK (mode IN ('full','incremental')),   -- full|incremental
  files_changed INTEGER NOT NULL DEFAULT 0,
  chunks_written INTEGER NOT NULL DEFAULT 0,
  ok INTEGER NOT NULL DEFAULT 0
);
```

`Store::open` applies `journal_mode`, `synchronous`, `mmap_size`, `busy_timeout` and `foreign_keys = ON` on every connection (an in-memory or pooled connection without them silently loses the CASCADE and FTS guarantees). Each index run ends with `PRAGMA optimize`. Disk pragmas cannot be mixed across connections sharing the WAL file — open once per process and reuse.

Row estimates at KanPad scale (723 files, ~190 tsx / 161 md / 118 json / 105 rs / 85 ts): ~4–6k chunks; vectors 768 float32 = 3 KB each ⇒ ~18 MB; FTS trigram ≈ 2–3× source bytes (~10–20 MB); total DB ≤ 150 MB (NFR-10).

### 4.3 Coverage invariant (AST chunks + gap spans)

Chunker emits **symbol chunks** (body + attached doc) and fills everything else with **gap chunks** (line windows ≤ 120 lines, split at blank lines when possible, **abutting — gap windows never overlap**). Every byte of every indexed file is in exactly one chunk (a *partition*, not merely a cover). Non-parseable/oversized files → pure line-window chunks with `kind='gap'`, `symbol=NULL`. This is what makes "finds relevant content without missing files" mechanically true — embeddings can't see bytes that aren't chunks.

Overlapping windows were rejected: double-indexed bytes break the exact-one property and make the reassembly test fail by construction. The 4-line overlap people want exists only in **snippet rendering** (a snippet may include ±4 lines of source context around its chunk) — never in stored coverage.

**Coverage property test (§9):** for random files/trees, chunks sorted by `start_byte` tile the file exactly — contiguous, non-overlapping, total. Comparison is on **source byte spans** (`start_byte`/`end_byte` slices of the original bytes), never on concatenated `text`, because each chunk's text carries a path/breadcrumb header that is not source bytes.

### 4.4 Concurrency model

- **Readers**: queries run on the calling thread against a WAL snapshot. They never block writers and writers never block readers. A query that arrives mid-reindex sees the pre-reindex state — acceptable and explicitly rate-limited by staleness note (US-104).
- **One writer process**: reindex/index takes `<db>.lock` via std `File::try_lock` (advisory, nonblocking, `flock` on Unix / `LockFileEx` on Windows). A concurrent server that loses the lock reports `reindex: busy` and serves stale results with a note.
- **Small write transactions**: per-file batch (delete rows for path → insert chunks → FTS/vec rows → commit) targets ≤ 200 ms txns; readers never see a half-written file's rows because delete+insert are atomic per file. FTS row maintenance rides the same transaction via triggers (schema.sql) — application code never maintains chunks_fts by hand, and FK enforcement (`foreign_keys = ON`, per connection) makes the CASCADE story airtight.
- **Reduced CPU priority**: indexing runs at background class. Linux/macOS: per-thread priority lowered (`nice(19)`; best-effort I/O-class reduction via `ioprio_set`/`setiopolicy_np` where available). Windows: `SetThreadPriority(THREAD_MODE_BACKGROUND_BEGIN)` on the indexing thread (CPU+I/O+memory drop to background band). All calls are best-effort: refusal is logged at debug, never fatal, and never blocks the run. The behavioral requirement — not the mechanism — is that the query path and the MCP event loop contain no indexing logic and never wait on it; indexing work enters only via an explicit `index`/`reindex` call. There is **no background indexing in v1**.
- **Memory**: prepared statements reused per connection; vector scan materializes only top-k (brute-force cosine is capped at chunk count, ~15 MB at 5k×768); no full-table loads into memory; RSS ≤ 100 MB (NFR-4).

## 5. Retrieval pipeline

```
query (string, optional filters)
  │
  ├─ if Ollama reachable (probe cached ≤30s): embed(query) ──► vec candidates:
  │      SELECT c.id AS chunk_id, vec_distance_cosine(v.embedding, ?) AS d
  │      FROM vec_chunks v JOIN chunks c ON c.id = v.chunk_id
  │      WHERE <filters — pre-fusion>  ORDER BY d LIMIT 32
  │
  └─ FTS candidates (always):
        SELECT c.id AS chunk_id, bm25(chunks_fts) AS s
        FROM chunks_fts JOIN chunks c ON c.id = chunks_fts.rowid
        WHERE chunks_fts MATCH <quoted-escaped query> AND <filters — pre-fusion>
        ORDER BY s LIMIT 32
  │
  │     filters (path_prefix / kind / lang) live INSIDE both candidate queries, so a
  │     filtered search returns real top-N rankings within the filter (US-203 AC4)
  │
  ├─ fusion: RRF k=60 over the two ranked lists (filters already applied)
  │     score(d) = Σ_lists 1 / (k + rank_l(d))   · sources weighted 1.0 (equal) in v1
  │
  ├─ boosts (post-fusion, MULTIPLICATIVE — they reorder near-ties, they never dominate):
  │     score'(d) = score(d) · (1 + w)
  │     symbol exact  w=+0.50 (×1.5) · symbol prefix w=+0.25 (×1.25), case-insensitive
  │     path component match  w=+0.10 (×1.1)
  │     recency (v0.3 via git churn)              w ≤ +0.05 (≤ ×1.05)
  │
  ├─ exact-identifier short-circuit: if query matches ^[A-Za-z_][A-Za-z0-9_]{2,}$
  │     AND FTS returns ≥ 3 hits ⇒ FTS-only ranking for those, vector appended after (tagged)
  │
  └─ shape: top `limit` (10) chunks → compact JSON (§6) with repo-relative paths,
       line spans, symbol/kind, fused score, snippet (≤ 20 lines, ≤ 2,000 tokens total),
       plus one-line staleness note when applicable
```

**Why multiplicative boosts (not additive):** RRF scores are small by construction — an item ranked #1 in both lists fuses to 2/(60+1) ≈ 0.033. An additive +0.50 boost would be ~15× the fused signal and would turn "symbol exact" into a ranking override that swamps fusion, contradicting both the fused-score intent and boosts-as-nudge. Multiplicative boosts scale with the fused score, leave non-matching candidates untouched (×1.0), bound the worst case (×1.5), and match PRD US-201 AC3 verbatim. Tie-break `(score, chunk_id)`.

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
       "created_at": "…", "updated_at": "…",
       "git_head_indexed": "…", "git_head_current": "…", "stale": false,
       "estimated_stale_files": 0,
       "embedder": { "model": "nomic-embed-text", "dims": 768, "reachable": true },
       "last_run": { "mode": "incremental", "finished_at": "…", "ok": true } }

// reindex
{ "mode"?: "incremental" | "full" }     // default incremental
→ { "mode": "incremental", "files_changed": 3, "chunks_written": 41,
    "duration_ms": 812, "ok": true }    // busy → { "ok": false, "reason": "busy" }, HTTP-free
```

The MCP `reindex` tool is the MCP face of the `index` CLI subcommand — same code path, same `<db>.lock`, same stats struct. The CLI `status` subcommand is a one-shot dump of `index_status` (single JSON object to stdout, not MCP-framed).

Error model: tool-level `isError` with short actionable message; never raw SQL/panics.

### 6.3 Cold-start inline index (US-301 AC3)

First `code_search` (or `get_file_outline`) on a server with no index performs a full index **inline** and answers in the same call. This is deliberate zero-config behavior (P1/P2) and is an explicit exception to per-call latency budgets — NFR-3's "index exists" scope note says so. When the client supplies an MCP `progressToken`, the server emits `notifications/progress` with `files_done`/`files_total` during the build; the final result reports `duration_ms`. If the process is killed or the client disconnects mid-build, per-file transactions plus the manifest reconcile on the next run (NFR-13): no half-written file state is ever visible, and nothing outside the index files is touched. Integrators who cannot tolerate a long first call should run `codebase-intel index` (or `reindex`) before their first search — that path is identical modulo the caller waiting.

### 6.4 stdout and logging discipline

In `serve` mode **stdout is the MCP protocol stream**: every line on stdout must parse as one JSON-RPC message. All logs go to stderr or to a file (`CODEBASE_INTEL_LOG=/path`). `tracing-subscriber` is initialized with `writer=stderr` in `main.rs`. No `print!`/`println!` anywhere in library code; `eprintln!` only in CLI one-shot output paths (never in `serve`). The one intentional JSON-to-stdout path is the `status` subcommand, which is not MCP-framed. Enforced by an integration test: spawn `serve`, complete `initialize`, assert every stdout line parses as JSON-RPC (§9).

## 7. Failure modes

| Failure | Behavior | Recovery |
|---|---|---|
| Ollama down/timeout | FTS-only results, `degraded: true` | automatic re-probe |
| Ollama wrong model dims | refuse vector ops; log; FTS-only until `reindex --full` re-embeds | `reindex --full` after pulling correct model |
| Index DB corrupt | open error → `index_status` reports `corrupt`; search refuses | `index --full` (deletes + rebuilds; safe) |
| Kill -9 mid-reindex | per-file txns ⇒ no partial file rows; manifest/index_runs truncate at last commit | next run reconciles by hash; `--full` if paranoid |
| Cold-start inline index killed (client timeout/kill) | identical to kill-mid-index: per-file txns, manifest reconciles; zero partial state | next call completes or re-triggers the build |
| Two servers reindex concurrently | loser: `busy`, serves stale + note | n/a |
| Disk full during index | txn rollback; `ok:false` with reason; old chunks for changed file intact (delete+insert atomic) | free space, re-run |
| File vanishes between walk & read | file skipped, counted in report, run continues | n/a |
| Fixture/test DB misuse | tests use tempdir DBs only; production path never touched in tests | enforced by `testutil` |

## 8. Security and privacy

- **Project-root confinement**: walker starts at the resolved project root; canonicalization rejects symlink escapes (a symlink pointing outside the root is skipped). `get_file_outline` / snippets never emit content from outside the root. **Tool-path confinement** mirrors the walker: every `path` argument (outline, search `path_prefix`) is normalized (repo-relative, `/` separators, leading `./` stripped), and any value resolving outside the canonical root — `..` escapes, absolute paths, symlinked parents — is rejected before any filesystem access or DB lookup.
- **Secret exclusion** (walker) — canonical list; PRD references this section, never restates it: `.env*`, `*.pem`, `*.key`, `*.p12`, `*.pfx`, `*.keystore`, `*.jks`, `id_rsa*`, `id_ed25519*`, `id_dsa*`, `id_ecdsa*`, `credentials*.json`, `secrets*.{yaml,yml,json,toml}`, `.npmrc`, `.pypirc`, `*.token`, `.envrc`, `.netrc`, `.git-credentials`, `*.tfvars`, `*.p8`, `*.ovpn`, `htpasswd`. Exclusion is name-based and applies **whether or not the file is tracked by git** (already-tracked secrets are the user's problem — still excluded by name). `.gitignore` / `.git/info/exclude` are honored first.
  - **Residual risk (documented, not silently accepted)**: name patterns cannot detect content-encoded secrets behind innocuous filenames — `.kube/config`-style kubeconfigs embedding base64 client keys, generic YAML/JSON with embedded tokens, `terraform.tfstate`. The remedy is the user's own `.gitignore` discipline; this is stated here so it is a known limitation, not an implied guarantee.
  - **Escape hatch (semantics provisional — see PRD Q4)**: `.codebase-intel-include` (repo-root dotfile, user-created — the tool never creates it) force-includes the repo-relative paths it lists. Provisional rule: it overrides ignore/class *size/type* filters only; secret-name exclusion and canonicalization confinement (inside root, no escapes) always win. Ratify or change in the v0.1 design review.
- **CI secret-pattern scan** covers `tests/fixtures/**` **and** any committed `docs/eval/*.md` (baseline reports contain identifiers copied from real repos — scan before commit, not after).
- **Network**: only `http://localhost:11434` (configurable via `CODEBASE_INTEL_OLLAMA_URL` to another *local or trusted* address; documented as such — code content would be sent there). No DNS, no telemetry, no update checks (NFR-8). CI grep-gate: no `reqwest` calls outside `embedder.rs` + `gitfetch.rs` (v0.3, none planned).
- **Telemetry**: none. Ever. `index_runs` stays local.
- **Phase-2 privacy**: session snapshots are copied with SQLite's `.backup` into a temp dir, mined offline, then deleted by default; mined JSONL for *private* use stays out of git (`.gitignore`d); public fixtures come from public repos only (eval-spec.md §7).
- **MCP trust boundary**: stdio local process inherits user privileges — like git. Server never writes outside `indexing` code path, and repo files are read-only to us.

## 9. Testing strategy

| Layer | What | Tooling |
|---|---|---|
| Unit | RRF fusion math, multiplicative-boost ranking, gap-span coverage invariant (byte-span tiling, no overlap), trigram escaping, canonicalization/symlink logic, config parsing | built-in `cargo test` |
| Property-ish | Random file corpus → chunk → chunks tile each file exactly: sorted by `start_byte`, contiguous, non-overlapping, total; byte-span comparison, headers stripped; 10k random trees | hand-rolled with seeded RNG |
| Integration | index fixture repo (mocked Ollama via `wiremock`) → search → assert expected files in top-5; degraded mode; reindex x2 idempotence; kill-mid-index → status → full rebuild; cold-start inline index (no index → `code_search` builds inline, reports `duration_ms`) | tempdir + wiremock |
| Retrieval quality | golden queries vs fixture repo; Recall@k/MRR floors; **fails CI on > 2% Recall@10 regression vs committed baseline** (after v0.2 defines fixtures) | `retrieval_quality.rs` |
| CLI contract | `index`, `status` output shapes parse as JSON; exit codes; `serve` emits only JSON-RPC frames on stdout | assert_cmd style + MCP stdio purity test |
| Perf smoke | index build < 60 s; no-change incremental < 1 s; query < 150 ms on fixture | feature-gated `bench` tests, run manually/weekly |

Platform-specific test policy: the symlink-escape walker test is `#[cfg(unix)]`-only (Windows symlink creation needs elevated privilege); file-lock tests run on all three OSes in CI; priority demotion is best-effort and asserted only "no stronger than normal" (never fatal on refusal). RSS checks use `sysinfo`.

Coverage floor: **80% lines** (`cargo llvm-cov`), enforced at release; unit+integration combined. Ollama-dependent tests are feature-gated (`--features live-ollama`) and excluded from default runs (NFR-9).

## 10. Build, release, compatibility

- Rust edition 2024; **MSRV = current stable − 2** (≥ 1.89 — required by stable file locking, `std::fs::File::try_lock`); toolchain pinned by `rust-toolchain.toml`
- Deps (first cut): `rmcp`, `rusqlite` (bundled), `sqlite-vec` (bundled, runtime `load_extension`), `tree-sitter` + grammars (rust, typescript, javascript, markdown, sql, yaml, toml, json), `ignore`, `blake3`, `reqwest` (rustls), `tokio`, `clap`, `gix`, `serde`/`serde_json`, `tracing`/`tracing-subscriber`, `thiserror`, `anyhow` (main only), `directories` (platform data dirs), `libc`/`windows-sys` (per-platform priority calls only, confined to `util.rs`); `sysinfo` as a dev-dependency (RSS test)
- Release: `cargo build --release` with `strip = true`, `lto = "thin"`, single codegen unit; `cargo-dist` for cross-builds (x86_64-unknown-linux-gnu, aarch64-unknown-linux-gnu, macOS arm64/x86_64, x86_64-pc-windows-msvc); crates.io publish as `codebase-intel`; npm wrapper (Q2) may follow
- DB schema migrations: none until v0.2+ if unavoidable; until then, `schema_version` bump ⇒ drop-and-rebuild (documented, rebuild is ≤ 1 min)
- Store hygiene: per-connection pragmas + `PRAGMA optimize` at end of index runs; `PRAGMA integrity_check` belongs to `index_status`'s corrupt-detection path — details in §4.2
- All three are first-class: CI runs fmt + clippy + tests on ubuntu-latest, macos-latest and windows-latest; release binaries ship for all three (NFR-12). Platform-specific code is confined to `util.rs` (paths, priority) and the walker (symlink handling) — everything else is platform-agnostic and any `#[cfg(unix)]`/`#[cfg(windows)]` outside those two fails the `deps` test (§11). Windows caveats, all documented: local NTFS only (WAL and mmap require local disks), filesystem is case-insensitive (repo paths stored exactly as walked; the `blake3(canonical_root)` hash is machine-specific — indexes are per-machine), long paths (>260) rely on std wide-character APIs (note `\\?`-verbatim UNC prefixing as follow-up if dogfooding demands it)

## 11. Module dependency rules

- `store` depends on nothing project-internal except `util`
- `retrieve` depends on `store`, `embedder`
- `mcp` depends on `retrieve`, `store`, `gitmeta`
- `indexer` (walker/chunker) depends on `store`, `embedder`, `util`
- **No cycles**; enforced by `cargo` module layering + a simple `deps` test listing allowed pairs, which also fails if platform `cfg` appears outside `util.rs`/`walker.rs`
- `eval` (v0.2) is a separate binary target sharing `store`/`retrieve` read-only — it must never write to the index
