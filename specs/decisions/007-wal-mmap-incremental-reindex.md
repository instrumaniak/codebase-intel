# ADR 007 — WAL + mmap + content-hash manifest for incremental reindex (no watcher)

## Status: Accepted

**Revision notes (2026-11-19, pre-implementation spec review):** (1) `O_EXLOCK` → `flock`; (2) incremental hashing unified to "hash every walked file" (the old "mtime prefilter" was a second, conflicting algorithm; blake3 at 1k-file scale is sub-second anyway); NFR-2 → US-302 AC2; (3) locking now std `File::try_lock` — one code path, advisory lock on all platforms — because Windows is first-class (ADR 009).

## Date: 2026-11

## Context

Indexes must reflect the working tree as it changes without (a) burdening active development/tests, (b) holding locks readers need, (c) requiring a watcher daemon (ADR 001), (d) missing deletions. Alternatives: filesystem watcher (inotify/FSEvents) in the server, mtime-only freshness, git-HEAD-only invalidation, periodic background re-scan inside `serve`.

## Decision

- **Invalidation is pull-based, not push-based**: freshness checked when someone calls search/status/reindex — compare `git HEAD` + per-file blake3 manifest; no background thread, no watcher, **0% idle CPU** (NFR-7). **Hashing covers every walked file** (one blake3 pass, sub-second at 1k-file scale); `mtime_ms` is stored for staleness notes but never gates hashing — there is no mtime-based second path.
- **Readers never block**: WAL snapshot reads; writers do ≤ 200 ms per-file delete+insert transactions (US-302 AC2).
- **mmap**: `PRAGMA mmap_size=256 MB` so hot pages (vectors, FTS index) are served from page cache without syscalls.
- **Indexing thread** (only when reindex runs): background-class priority (Unix: per-thread `nice(19)`; Windows: `SetThreadPriority(THREAD_MODE_BACKGROUND_BEGIN)`); chunk-level batches (32) with brief sleeps; respects `reindex --full` as the escape hatch. Behavioral requirement: the query path never waits on indexing.
- **Serialization**: `<db>.lock` via std `File::try_lock` (nonblocking; advisory; one code path on all platforms) — concurrent reindex attempts lose gracefully with `busy`. Lock released on handle close (process exit); the lock file may persist as a sentinel.

## Reasons

### Why not a watcher
- A watcher in a short-lived stdio server sees almost nothing: it exists only while a client session lives and the editor/build system hammers the tree. Correctness under bursts (10k mtime events from `cargo clean`) demands coalescing + debounce — complexity to deliver a freshness property the manifest already provides for free at query time.
- Watchers make "index is disposable" murkier: stale inotify state across unlink/rewrite cycles (rustc swaps rlibs by rename!) is a classic source of phantom entries.

### Why content hash over mtime
- Git ops (checkout, stash, rebase) touch mtimes without content change; hashing is immune. Cost is one blake3 pass over **every walked file** (~µs per small file, sub-second total at target scale) — hashing everything is simpler and eliminates the changed-mtime fast path that would drift from the one true change detector.
- `Cargo.lock` churn, editor swap files (`*.swp`), and `target/` are filtered before hashing (walker filters + `.gitignore`), so hashing noise is bounded.

### Why not git-HEAD-only invalidation
- Working tree edits (the common case during a session) don't move HEAD. Manifest catches those; HEAD catches branch switches; both needed, neither sufficient alone.

### Why no periodic background scan inside `serve`
- Would violate NFR-7 (idle CPU) and surprise users on battery/CI machines. Explicit beats clever here: the agent (or human) decides when to reindex; `index_status` nudges when stale.

## Consequences

- Worst-case query on a very stale index costs one manifest re-check (fast) + possibly a staleness *note*, not a silent wrong answer; agent instructions say when to call `reindex`.
- Two servers reindexing concurrently: one waits/loses by design; no corrupt merge risk because per-file transactions are atomic and full rebuild is `DELETE all + rebuild`.
- mmap on 32-bit is absent — 32-bit targets remain unsupported on all operating systems; 64-bit x86_64 and aarch64 Linux/macOS/Windows are tier-1.

## Revisit Trigger

- Dogfood shows staleness confusion (agents searching stale code > 2× per week) → add opt-in `serve --index-on-start=if-stale` (would be a new, off-by-default flag — not in v0.1 scope), not a watcher.
- Manifest check grows past ~50 ms on very large trees — shard by directory or persist directory-level hashes.
- macOS FSEvents behavior diverges enough to matter — then revisit with data, not preference.
