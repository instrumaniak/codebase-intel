# Epics & release phasing — codebase-intel

**Status:** v1 draft — awaiting owner review
**Companion docs:** [prd.md](prd.md) (US IDs referenced below) · [architecture.md](architecture.md) · [eval-spec.md](eval-spec.md)
**Note:** Epics here feed the *later* taskpad breakdown. **No task files exist yet.**

## Phasing at a glance

| Phase | Version | Theme | Exit gate |
|---|---|---|---|
| 1 | v0.1.0 | Code search that works, on your machine | S1/S5 dogfood on KanPad; all v0.1 ACs green |
| 2 | v0.2.0 | Measured retrieval quality | Baseline + tuning report committed; S2/S3 |
| 3 | v0.3.0 | Why-did-this-change (git intel) | US-501 ACs green |
| 4 | v0.4.0 | Open-source release | S6; crates.io; docs live |

---

## E1 — Indexer core (v0.1)

**Goal:** repo → SQLite index, reliably, fast, ignoring junk.

| Story | Summary | Delivers |
|---|---|---|
| US-101 | Zero-config index of a git repo | F1 |
| US-102 | Incremental reindex by content hash | F3, F1 |
| US-103 | AST chunks + gap-span coverage | F2 |
| US-104 | Staleness detection & reporting | F3, F6 |
| US-105 | Degraded mode when Ollama is down | F4 |

**Epic exit:** `codebase-intel index` on the KanPad repo produces a queryable DB; deleting/modifying files re-indexes only those; killing mid-index recovers on next run.

## E2 — Retrieval (v0.1)

**Goal:** "describe what it does" → the right files, compact output.

| Story | Summary | Delivers |
|---|---|---|
| US-201 | Hybrid search (dense + FTS → RRF) | F5 |
| US-202 | Identifiers stay reliable | F5 |
| US-203 | path/kind/lang filters at SQL layer | F5 |
| US-204 | Compact, token-budgeted results | F6 |
| US-205 | File outline without reading the file | F6 |

**Epic exit:** 10-query dogfood spot check finds the intended file ≥ 9/10 times (S1).

## E3 — MCP surface & lifecycle (v0.1)

**Goal:** invisible in daily work; safe next to builds and tests.

| Story | Summary | Delivers |
|---|---|---|
| US-301 | Zero-config MCP launch + instructions | F6, F7 |
| US-302 | Low-priority background indexing, WAL reads, lock-file serialization | F3, F6 |
| US-303 | Resource appetite within budget | F6 |

**Epic exit:** opencode with one config entry spawns the server; first query on a cold index triggers the inline full index (US-301 AC3) and answers in the same call with `duration_ms` reported (KanPad-scale: minutes, embedding-bound; progress via MCP `notifications/progress`); `cargo test` on KanPad shows no measurable slowdown while reindex runs.

## E4 — Evaluation (v0.2)

**Goal:** turn session history into numbers; tune only what the numbers endorse.

| Story | Summary | Delivers |
|---|---|---|
| US-401 | Offline session miner (snapshot → JSONL) | F8 |
| US-402 | Ground-truth derivation + dedup + holdout | F9 |
| US-403 | Metrics + baseline report | F9 |
| US-404 | Tuning loop + before/after documentation | F10 |

**Epic exit:** baseline report committed; one tuning iteration completed end-to-end; CI retrieval gate exists (fixtures permitting); S2/S3.

## E5 — Git intelligence (v0.3)

**Goal:** "why is this code like this" — commit search + churn-aware ranking.

| Story | Summary | Delivers |
|---|---|---|
| US-501 | `git_search` (semantic commits → touched files) | F11 |

**Epic exit:** a real "why did X change" question answered from this repo's 460-commit history; churn boost measured against eval set (≤ 5% contribution).

## E6 — OSS release (v0.4)

**Goal:** a stranger installs and uses it without meeting the author.

| Story | Summary | Delivers |
|---|---|---|
| US-601 | cargo install + release binaries + docs + CI | F12 |

**Epic exit:** `cargo install codebase-intel && codebase-intel index && echo {"jsonrpc":…} | codebase-intel` round-trips on a clean machine; README quickstart verified verbatim in a fresh container; S6.

---

## Dependency graph

```
F1 ──▶ F2 ──▶ F3 ──▶ F5 ──▶ F6 ──▶ F7 ──▶ (v0.1 dogfood) ──▶ F8 ──▶ F9 ──▶ F10 ──▶ F11 ──▶ F12
        │      │      │      │
        └─▶ F4 ───────┘      └─▶ (parallel with F4)
```

## Task breakdown guidance (for later, via taskpad)

When specs are approved, tasks are authored per epic in dependency order, one PR-sized task each, each with Goal / Steps / Depends-on / Acceptance (KanPad house style). Rough task-count expectation, not a commitment: E1 ≈ 6–8 tasks (walker, grammar dispatch, chunker+gaps, store+schema, embedder+degraded, incremental manifest), E2 ≈ 4–5, E3 ≈ 3–4, E4 ≈ 4–6, E5 ≈ 2–3, E6 ≈ 3–5. ~25 tasks total to v0.4 — deliberately not exploded further until the spec review passes.
