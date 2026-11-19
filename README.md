# codebase-intel

**Local-first, AST-aware semantic code search for AI coding agents — over MCP.**

`codebase-intel` is a stdio [Model Context Protocol](https://modelcontextprotocol.io/) server that indexes your repository — AST-aware chunks, FTS5 keyword index, and local embeddings — and answers hybrid search queries with compact, line-referenced snippets. It exists because agents that only have `grep`/`glob` miss files whose *meaning* matches the task but whose *spelling* doesn't, and then edit the wrong file or reimplement what already exists.

- **Hybrid retrieval** — dense vector similarity (local Ollama embeddings) fused with BM25 keyword search (SQLite FTS5) via Reciprocal Rank Fusion. No reranker in v1 — deliberately.
- **AST-aware chunking** — tree-sitter extracts symbols (functions, classes, impls, methods) with their doc comments; unfilled regions are still indexed, so *no source bytes are invisible*.
- **Lightweight & background-proof** — single binary, no daemon, ~<100 MB RAM, queries in <150 ms warm, incremental reindex that never blocks you (or your test suite) on a WAL snapshot.
- **100% local** — embeddings via your own Ollama on localhost; nothing leaves the machine; no telemetry, no API keys.
- **Session-tuned (planned)** — retrieval quality is *evaluated* against real agent session history (what the agent searched, which files it actually edited), not vibes. See `specs/planning-artifacts/eval-spec.md`.

> **Status: pre-release. Spec stage.** Nothing here is implemented yet — the specs in [`specs/`](specs/) are the current source of truth. See the [roadmap](specs/planning-artifacts/epics.md).

## Why not just grep?

Agents do reach for grep — and grep is exact. The misses are the files that match *conceptually*:

```
Agent task: "fix the auth token expiry bug"
Files:      src/auth/mod.rs          ← grep finds this
            src/session/refresh.rs   ← grep misses (no 'auth' in name or text match that matters)
            docs/token-lifecycle.md  ← grep misses (conceptual match)
```

`codebase_intel.code_search("fix auth token expiry bug")` is designed to return all three.

## How it works

```
source tree ──▶ walker (gitignore-aware) ──▶ tree-sitter chunks ──▶ ┌─ files        (mtime/hash)
              (skip junk, size caps)                               ├─ chunks        (FTS5 + vec)
                                                                   └─ meta          (git HEAD, model)
agent query ─▶ embed (Ollama, local) ─┬─▶ vector top-32 ─┐
                                      └─────▶  FTS5 top-32 ─┴─▶ RRF fusion ─▶ compact snippets (path:lines)
```

## Install

Not yet released. Build from source:

```bash
git clone <repo-url> && cd codebase-intel
cargo build --release
```

Register with opencode (`~/.config/opencode/opencode.json` or `.opencode/opencode.json`):

```jsonc
{
  "mcp": {
    "servers": {
      "codebase-intel": {
        "type": "local",
        "command": ["/path/to/codebase-intel"]
      }
    }
  }
}
```

Prereqs: [Ollama](https://ollama.com/) with `nomic-embed-text` pulled (`ollama pull nomic-embed-text`). Degraded mode without Ollama: keyword-only search, still useful.

## Tools

| Tool | Purpose |
|---|---|
| `code_search` | Hybrid semantic + keyword search over the index |
| `get_file_outline` | Symbols + line ranges of one file (no bodies) |
| `index_status` | Index age, coverage, embedder health, what's stale |
| `reindex` | Incremental by default, `--full` available |

## Design notes

- **No daemon.** The server is spawned by the MCP client per session; the index lives in `~/.local/share/codebase-intel/`. WAL mode means readers never block.
- **No reranker in v1.** Evidence: cross-encoder reranking costs ~90 s/query on CPU and lost to plain dense retrieval with a good embedder. RRF + symbol boosts is the v1 bet (see [ADR 004](specs/decisions/004-rrf-hybrid-retrieval-no-reranker-v1.md)).
- **The index is disposable.** Worst case, `codebase-intel index --full` rebuilds it in under a minute on a ~1k-file repo.

## License

Dual-licensed under [MIT](LICENSE-MIT) or [Apache-2.0](LICENSE-APACHE), at your option.
