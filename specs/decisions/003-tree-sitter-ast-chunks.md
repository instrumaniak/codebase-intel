# ADR 003 — tree-sitter AST chunks with gap-span coverage

## Status: Accepted

## Date: 2026-11

## Context

Retrieval quality is bounded by chunking. A function split across two chunks is unfindable as a unit; a function glued to three unrelated symbols embeds as semantic soup; and any region *not* covered by chunks cannot be retrieved at all (the tool's core claim depends on complete coverage). Alternatives: fixed line windows, regex-based "function splitting", language-specific parsers (rustc, TypeScript compiler API).

## Decision

tree-sitter with per-language symbol queries for supported languages (v1: Rust, TS/TSX, JS/JSX, Markdown, SQL, YAML, TOML, JSON), plus three invariants:

1. **Symbol chunks**: named nodes (fn/impl/trait/struct/class/…) with attached doc comments, including a breadcrumb (`parent`) of enclosing symbols.
2. **Gap chunks**: all uncovered bytes are indexed as line-based chunks (≤ 120 lines, blank-line preferred splits, 4-line overlap).
3. **Every byte is in exactly one chunk.**

Unsupported/oversized files fall back to pure line-window chunking with the same coverage guarantee.

## Reasons

### Why AST over windows
- A window of "lines 200–260" may start mid-body and end mid-signature — the embedding then represents nothing coherent. Whole symbols with doc comments embed well.
- Symbols give retrieval *structure*: symbol-exact boost (ADR 004) needs `symbol` metadata; file outlines (US-205) need it; churn joining (v0.3) needs it.
- tree-sitter parses 50+ languages from small WASM/native grammars without invoking each language's toolchain (no rustc/tsc required at index time) — critical for "index any repo in < 60 s".

### Why gap spans (the under-appreciated part)
- Magic constants, SQL strings, config blobs, markdown prose, module-level statements — semantically meaningful content that symbol grammars deliberately skip. Keyword search can still find a literal string in a gap chunk; semantic search gets a window that at least isn't truncated mid-symbol.
- Coverage converts "search didn't miss the file" from aspiration to **invariant with a property test** (reassemble all chunks == original bytes, for random trees).

### Why not language-native parsers
- rustc/TS-compiler coupling means indexing depends on project toolchain availability and version parity; tree-sitter is syntax-only, fast (~10–50 MB/s), and uniform.

## Consequences

- Grammar maintenance is ongoing (TS/TSX quirks, MD embedding code blocks). Gap spans bound the blast radius — a bad grammar never makes content invisible, only suboptimal.
- Doc attachment rules are language-specific (JSDoc/doc comments/preceding comments within N lines); each is a small query set in `chunker/treesitter.rs`.
- Larger chunks cost more embedding time; oversized symbols (> embedder budget) are truncated with explicit marker — never silently dropped.

## Revisit Trigger

- A retrieval bucket underperforming because of chunking (identified via eval-spec.md §5/§6) — e.g., "conceptual over config docs" may want document-structure chunking for Markdown (headers as parent, not file).
- tree-sitter mis-parses a critical grammar beyond repair → regex-fallback for that language behind the same coverage invariant.
