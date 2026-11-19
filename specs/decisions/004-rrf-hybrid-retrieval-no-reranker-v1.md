# ADR 004 — Hybrid dense+FTS with Reciprocal Rank Fusion; no reranker in v1

## Status: Accepted

## Date: 2026-11

## Context

Given chunk vectors and an FTS5 inverted index, how do we rank? Options: dense-only, sparse-only, RRF fusion, learned fusion (linear/vector), cross-encoder reranking, MMR diversification. Available evidence from adjacent tools: a published agent who enabled Qwen3-Reranker-0.6B found ~90 s/query on CPU and *lower* scores than plain dense retrieval with a strong embedder; and the calling agent re-ranks results anyway when reading snippets.

## Decision

v1 retrieval = **vector top-32 (cosine, exact) ∪ FTS5/BM25 top-32 → Reciprocal Rank Fusion (k = 60, equal weights) → tiny post-fusion boosts** (symbol exact ×~+0.50 additive, symbol prefix +0.25, path-component +0.10, recency ≤ +0.05 in v0.3). **No cross-encoder reranker. No learned fusion weights. No MMR.**

## Reasons

### RRF is cheap, robust, and unglamorous
- RRF composes ranked lists without calibrating score magnitudes (cosine distance vs BM25 are incomparable units — RRF sidesteps this by using ranks only).
- k = 60 is the literature default; list size 32 each keeps candidate gen ~2× cheap SQL (indexed cosine over 5k rows; trigram FTS) — comfortably inside NFR-1.
- Deterministic (tie-break by chunk id) — required for eval determinism (NFR-11).

### Dense-only fails exactly the identifiers we care about
- Code search queries are often literal tokens (`x-forwarded-for`, `ValidJson`, `refresh_token`). Dense embeddings underperform here; BM25 owns it. The hybrid covers both without explicit query classification.

### FTS-only fails paraphrase and synonyms
- "token expiry" vs "refresh_token TTL 60" — only the dense side bridges this. RRF lets each list contribute what it's good at.

### Why not a reranker (measured, not aesthetic)
- 0.6B cross-encoder ≈ 90 s/query on CPU — blows NFR-1/NFR-2 by 150×. GPU required, violating "invisible beside dev".
- The evidence says it *didn't beat* dense retrieval anyway with a decent embedder; adding latency to lose accuracy is indefensible.
- The calling agent is already a reranker: it reads the 5–10 snippets and re-ranks them cognitively for the actual task. Over-paying compute to pre-rank for an already-capable consumer is waste.

### Why not learned/MMR fusion
- Learned weights need labeled data we don't have yet (v0.2 eval set is the right substrate — revisit then). MMR buys diversity at the cost of a second relevance pass; snippets ≤ 20 lines each are cheap enough that near-duplicates are mostly harmless.

## Consequences

- Expect ~70–85% of eventual quality at v0.1; the eval harness (v0.2) exists to find the missing 15–30% systematically rather than by reranker superstition.
- Boosts must stay small — they break ties and nudge, never dominate; if a boost exceeds ~0.5 additive, it's a ranking smell and eval-spec §6 must measure it.
- Identifier short-circuit is a strict exception, documented in architecture.md §5.

## Revisit Trigger

- Eval harness shows FTS-only beats hybrid on a bucket (then drop dense for that bucket via short-circuit widening — data decides).
- A small (< 200 MB) local reranker demonstrably beats RRF by > 5% Recall@10 within latency budget on this repo family.
- Labeled pairs exist for learned fusion (post-v0.2).
