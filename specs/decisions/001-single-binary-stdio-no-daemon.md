# ADR 001 — Single binary, stdio MCP, no daemon

## Status: Accepted

**Revision notes (2026-11-19, pre-implementation spec review):** (1) lock mechanism corrected from `O_EXLOCK` (BSD/macOS only) to `flock`; (2) locking now one std code path — `File::try_lock` — because Windows is a first-class target (ADR 009).

## Date: 2026-11

## Context

The tool must be usable by MCP clients (opencode first) and must be "invisible in daily development — builds and tests run beside it". Options: (a) stdio server spawned per client session, (b) long-lived daemon on a fixed port, (c) editor plugin / CLI one-shot per query.

## Decision

Single `codebase-intel` binary; MCP clients spawn it on demand over **stdio**; no daemon, no port, no background process after the client exits.

## Reasons

### Cost model is right for this product
- MCP stdio is the lowest-friction transport with opencode and Claude Code; zero configuration, no discovery, no auth (trust boundary = parent process).
- A daemon solves a problem we don't have (sub-100 ms query latency doesn't need warm service amortization across sessions — our index is mmap'd and queries are ~100 ms).
- A daemon introduces lifecycle management (install/uninstall, crash restart, port allocation, logs) that directly competes with the "invisible in daily use" requirement.

### Failure isolation
- A crashed stdio server affects one tool call, not a shared service. opencode already handles per-server restarts.
- No socket, no firewall prompts, no "is the daemon running?" support burden for open-source users.

### Performance is sufficient without it
- Cold start ≈ open SQLite + read `meta` ≈ tens of ms; the expensive work (embedding) is per-query, not per-server.
- Index rebuilds are explicit (`index`, `reindex`), not triggered by server spawn — spawn latency stays trivial.

### Operational honesty for OSS
- "Works out of the box on a clean machine with only Ollama" is a stronger release story than "start this daemon first".

## Consequences

- Two clients spawning the server concurrently both work for **reads**; reindex serializes via `<db>.lock` (std `File::try_lock` — `flock` on Unix, `LockFileEx` on Windows), loser reports `busy` (see architecture.md §4.4, ADR 009).
- No push notifications (e.g., "index finished") — clients poll `index_status`. Acceptable.
- If future clients need HTTP transport, a `serve --http :8787` subcommand can be added later; the stdio path stays canonical.

## Revisit Trigger

- A client that cannot spawn subprocesses (web-only MCP endpoints) requires HTTP.
- Index open time grows past ~200 ms on very large repos — consider keeping the index warm in a shared cache process (but NFR-1 already forbids this becoming necessary).
