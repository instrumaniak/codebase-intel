# ADR 005 — Ollama HTTP embedder with nomic-embed-text (768-d)

## Status: Accepted

## Date: 2026-11

## Context

Chunks (~5k now, 50k ceiling) need float32 embeddings computable locally, offline, with acceptable quality on code + mixed markdown. Candidate backends: (a) Ollama `/api/embed` with nomic-embed-text, (b) in-process ONNX runtime (fastembed-rs, ort crate) with bge-code-v1 / bge-small, (c) OpenAI/Voyage/Cohere API embeddings, (d) qwen3-embedding via Ollama. Constraints: no cloud calls (privacy), no heavy runtime bundled in the binary, quality measurable via the v0.2 eval harness.

## Decision

**Default embedder = Ollama HTTP at `http://localhost:11434`, model `nomic-embed-text` (768-d, ~274 MB).** Behind an `Embedder` trait (`embed(&[&str]) -> Result<Vec<Vec<f32>>>`) so it is swappable without touching retrieval. Embedding batch = 32; circuit breaker: 3 consecutive failures/timeouts ⇒ degrade to FTS-only for 30 s (US-105).

## Reasons

### Why Ollama first
- Already the owner's stack (Cloud models installed; local machine runs it); zero new infrastructure; `ollama pull` is the entire install.
- CPU-only embedding of nomic-embed-text is fast enough for 5k chunks (~seconds); GPU unaffected.
- HTTP boundary keeps the binary small and avoids ONNX/ort native-build matrix pain pre-1.0 (one less thing to break on macOS CI).

### Why nomic-embed-text
- 768-d (small index; 3 KB/chunk), 8192-token context (chunks are ≤ ~150 lines), solid general text + decent code, smallest credible local option, best-documented Ollama path.
- **Not because it's optimal** — explicitly *unproven for code semantics until eval numbers say so*. That is what the harness is for.

### Why not API embeddings
- Non-negotiable privacy: code never leaves the machine (NFR-8); keys/accounts/failure modes rejected by design.

### Why not in-process ONNX first
- Binary size + ort/fastembed native packaging vs "Ollama already exists on the target machine". The trait leaves the door open (fastembed-rs is the credible in-process fallback; bge-small-en-v1.5 ~130 MB).
- If dogfooding shows Ollama embed latency blocking queries badly on cold model (first call ~1–2 s), revisit — the circuit breaker already prevents that from being a hard failure.

### Why not qwen3-embedding:0.6b immediately
- 1024-d doubles vector storage; gains on MTEB/code benchmarks are real but this repo's eval will tell. Track documented as the first embedder experiment post-harness (one line in tuning-log).

## Consequences

- Users without Ollama get keyword search only — documented, first-class, tested (degraded-mode integration test).
- Embedding model is recorded in `meta` (`embed_model`, `embed_dims`); model/dims mismatch on open refuses vector ops with a clear error and a `reindex --full` instruction — never silently mixed dimensions.
- Ollama process lifecycle belongs to the user (it is their machine tool); our code probes and adapts, never manages it.

## Revisit Trigger

- Eval-spec.md §6: hybrid-with-nomic Recall@10 loses to hybrid-with-qwen3 by > 5% on holdout — swap default.
- Users widely report Ollama-less setups; ship fastembed-rs as an optional `--features onnx` build.
- nomic-embed-text deprecated/pulled from Ollama registry.
