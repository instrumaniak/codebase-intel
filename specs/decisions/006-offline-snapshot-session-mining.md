# ADR 006 — Session mining reads an offline snapshot, never the live DB

## Status: Accepted

## Date: 2026-11

## Context

Phase 2 derives retrieval ground truth from opencode session history stored in a live SQLite WAL database (`~/.local/share/opencode/opencode.db`, opencode v2.3.2). Access options: (a) open live DB read-only, (b) SQLite `.backup` snapshot → mine the copy, (c) opencode HTTP API export per session (`/api/experimental/session/{id}/export`), (d) parse `~/.local/share/opencode/log/`.

## Decision

**(b) always.** Miner requires a snapshot path (or creates one itself via `.backup` into a temp dir and deletes it after); `PRAGMA query_only = ON` on the read connection. The live DB is never opened by the miner. The HTTP API is rejected for bulk mining.

## Reasons

### Correctness on a moving target
- A live read-only connection still sees uncommitted-then-committed WAL state mid-analysis; a turn mined as "no edits yet" could flip after the fact. `.backup` gives a consistent point-in-time image, making mines **reproducible** — a hard requirement when eval numbers are quoted in the project's claims and CI gate.

### Safety & hygiene
- Snapshots are disposable; deleting them post-mine is the default (US-401/AC3). No chance of corrupting a 237 MB live database that holds the owner's full working history — SQLite read-only is *usually* safe, but WAL quirks and V2 schema drift have bitten people before (opencode ships `OPENCODE_DB` overrides and warns against external writes).

### Why not the API
- 419 sessions × N messages → hundreds of HTTP round-trips; then we still don't get the exact row-level guarantees; and we'd couple eval tooling to a running service (ADR 001 spirit). The API is the right interface for *live session tooling*, the wrong one for offline analytics.

### Why not logs
- Process logs (`opencode.log`) are diagnostic, unstructured, and incomplete for this purpose; explicitly rejected as a data source.

## Consequences

- Miner is a separate subcommand (`mine-sessions`) that refuses to run against the live path (checks the snapshot is a *copy*: path ≠ resolved `OPENCODE_DB`, plus a `query_only` pragma).
- Snapshot inherits file permissions; mining temp dirs are created `0700` under XDG cache; no snapshot ever committed (`.gitignore`).
- Freshness ceiling = snapshot moment; fine for offline eval (history is history).

## Revisit Trigger

- If we later build *live* session tools (a real differentiator would be cross-session recall), design a separate read path with its own ADR — do not weaken this one.
- opencode V3 changes table layout: miner gains schema detection (it already is defensive); failure mode is "clear error", never "garbage labels".
