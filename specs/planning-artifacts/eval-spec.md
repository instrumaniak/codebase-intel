# Evaluation specification — codebase-intel

**Status:** v1 draft — awaiting owner review (this is Phase 2 of the roadmap; implemented after v0.1 ships)
**Purpose:** use opencode session history to (1) understand what kinds of queries agents actually issue, (2) build ground-truth retrieval evaluation sets, (3) tune ranking with measurement instead of vibes.
**Companion docs:** [prd.md](prd.md) (US-401..404, F8–F10) · [architecture.md](architecture.md) §5 · [epics.md](epics.md)

---

## 1. Why this exists

Every existing semantic-code-search MCP server ships ranking tuned by its author's taste. This project's claim is narrower and testable: **ranking choices for agent-facing code retrieval can be tuned against what agents actually did** — what they searched, what they opened, what they changed. If we can't measure that the tool beats grep for a real task, it has no reason to exist.

This document defines the data, the extraction, the labels, the metrics, the tuning loop, and the privacy rules. It is deliberately separate from the PRD because it will be iterated on independently.

## 2. Data source: opencode session storage (v2.3.2)

Facts established on the author machine (opencode v2.3.2, Linux, 2026-11-19 — re-verify at implementation time; the miner is defensive against schema drift):

- **Database:** `~/.local/share/opencode/opencode.db` (SQLite; `OPENCODE_DB` env var overrides). Live WAL database — never opened directly for mining.
- **Tables used:**
  - `session_v2` — one row per session: `id`, `project_id`, `directory`, `path`, `title`, `agent`, `model`, `parent_id` (subagent sessions), `time_created/updated`
  - `session_message` — one row per message: `id`, `session_id`, `type`, `seq`, `time_created`, `data` (JSON)
  - `project` — `id`, `worktree` (maps sessions to repos)
- **`session_message.data` shapes observed:**
  - `user` → `{ time, text, files[], agents[] }` — `text` is the task
  - `assistant` → `{ time, agent, model, content[] }` where content items are `reasoning` | `text` | `tool`
  - `tool` item → `{ id, name, input, state: { status, input, output } }`
  - Tool names observed (frequency): `shell` (13.3k), `read` (4.3k), `edit` (3.4k), `grep` (921), `write` (603), `websearch` (351), `execute` (291), `glob` (267), `subagent` (233), `webfetch` (166), `question` (79), `bash` (79)
  - Other types present: `system`, `synthetic`, `compaction`, `idle`, `shell`, `agent-switched`, `model-switched`, `location-switched` — ignored except `compaction` (aware-of, not mined)
- **observed install sizes:** ~419 sessions, ~24k messages, ~175 MB of `data` JSON, max single message ~515 KB, `session_part`/`event` tables empty on this install (do not rely on them).
- **logs** (`~/.local/share/opencode/log/opencode.log`) are **not** mined — they are process logs, not task context.

### 2.1 Snapshot protocol (mandatory)

1. `sqlite3 "$OPENCODE_DB" ".backup '<tmpdir>/snapshot.db'"` — atomic, consistent, works on a live WAL database (equivalent to SQLite `backup` API).
2. Mine only from the snapshot. Set `PRAGMA query_only = ON` on the read connection.
3. Delete the snapshot when mining finishes unless `--keep-snapshot` is passed.
4. Never open the real DB path from the miner without the `.backup` step; never write to it.

### 2.2 Session selection

- Target repo = canonical project root passed as `--project <path>` (e.g., `/mnt/mint/home/raziur/CodeLab/Personal/AITestLab/KanPad`).
- A session belongs to the project iff `session_v2.directory` is equal to or under that root (paths normalized), **or** the joined `project.worktree` matches. Both checks are applied; disagreement is reported, not silently resolved.
- Subagent sessions (`parent_id` not null) are mined and attributed to the parent task, flagged `is_subagent: true` — they are short, focused, and excellent query examples.
- Sessions with fewer than 2 user turns or zero tool calls are dropped (nothing to evaluate).
- Only files that exist **within the project root** are kept (the agent's `read`/`edit` paths are made repo-relative and prefix-checked).

### 2.3 Turn extraction

A **turn** = sequence of messages from one `user`-typed message (inclusive; use top-level `user` only, not `synthetic`) up to the next `user` message of the same session. Assistant `tool` content items in the turn are mapped:

| Field | Extraction |
|---|---|
| `task_text` | `user.data.text` (raw; length-capped at 4 KB, truncated with marker) |
| `greps[]` | from `tool name ∈ {grep}`: `{ pattern: input.pattern /* raw */, path: input.path }`; `globs[]` analog |
| `reads[]` | `files` in `tool name == "read"` → `input.path` |
| `edits[]` | `files` in `tool name ∈ {edit, write}` → `input.path` (for `write` of new files: keep, they are positives for "where would this new file go" queries) |
| `websearches[]` | `input.query` — used for taxonomy only |
| `zero_hit` | `output` text of grep/glob matches a "no matches" heuristic — **flagged, never hard-labeled**; exact phrasing must be verified empirically at implementation time (F8 includes this verification spike) |

Output JSONL (`eval.jsonl`), one record per turn:

```jsonc
{ "session_id": "ses_…", "turn": 7, "is_subagent": false,
  "task_text": "…", "greps": [{"pattern": "x-forwarded-for", "path": "src"}],
  "globs": [], "reads": ["backend-rs/src/common/extract.rs"],
  "edits": ["backend-rs/src/error.rs"],
  "websearches": [], "zero_hit": ["src/proxy"] }
```

## 3. Ground-truth labels

| Label | Source | Strength | Used for |
|---|---|---|---|
| `edited` | files in `edits[]` | strong positive (grade 3) | Metrics primary target |
| `read` | files in `reads[]` ∖ edited | medium positive (grade 2) | Metrics secondary target |
| `grep_hit` | files matching a non-zero-hit grep, never read/edited | weak positive (grade 1) | Diagnostic only, excluded from Recall/MRR computation |
| `zero_hit_task` | task had ≥ 1 zero-hit search | hard-example marker | Stratification, never a positive |

Rationale: a file the agent edited is *certainly* relevant (or the edit was necessary); a file it read is probably relevant (browsing or verifying); a file only grep-hit is noise. Negative sampling (files never touched in the turn) comes free from the ranked list itself — evaluation only needs positives.

Deduplication: drop near-duplicate tasks (trigram Jaccard of `task_text` ≥ 0.8). Sample ≤ 100 turns per project, stratified by query-type bucket (§5) and by phase of work, plus all zero-hit turns (they are rare and precious). Hold out 20% as a blind set used only for final checks.

## 4. Metrics and baseline

Grade values feed nDCG; binary dropped-label set feeds Recall@k and MRR.

- **Recall@1 / @5 / @10** — ≥1 `edited` file in the top-k (primary metric: Recall@10)
- **MRR** — reciprocal rank of first `edited` file (0 if absent in top-25)
- **nDCG@10** — graded: edited=3, read=2, grep_hit=1, else 0

Reported **overall** and **per query-type bucket**. Baselines that must be captured before any tuning (F9/US-403):

1. `grep-only` — rank files by "matched a grep the agent ran, first match wins" (approximation of agent behavior)
2. `dense-only` — vector retrieval, no FTS
3. `fts-only` — keyword retrieval, no vectors
4. `hybrid-rrf` — current default (k=60, no boosts beyond symbol-boost)

Determinism: same index + same eval set ⇒ identical numbers (tie-breaking by chunk id; eval never reads the clock).

## 5. Query-type taxonomy

Heuristic classification of `task_text` (documented rules, applied at report time, never inside the ranker):

| Bucket | Rule of thumb | Example |
|---|---|---|
| `identifier` | contains a camelCase/PascalCase/snake_case token ≥ 1 | "where is `handleJwtRefresh` used" |
| `error_message` | contains error-ish tokens (`error`, `exception`, `panic`, `failed`, `timeout`, `404`, `500`, stack-frame-ish path) | "login 429s everyone" |
| `config_ops` | mentions config files/ops topics (`cors`, `env`, `docker`, `ci`, `proxy`, `migrate`) | "fix CORS for the SPA" |
| `refactor` | contains `refactor|rename|move|delete|extract|clean` | "extract the retry helper" |
| `conceptual` | everything else | "how does session persistence work" |

The report (per eval run) prints: bucket sizes, and Recall@10 per bucket — tuning effort is then directed at the worst bucket, explicitly, in the commit message.

## 6. The tuning loop (US-404)

1. Capture baseline (dense/fts/hybrid) → `docs/eval/baseline-<date>.md`
2. Change **one** thing: chunk size, gap overlap, RRF k, boost weights, FTS tokenizer (trigram vs unicode61), symbol-boost magnitude, graph-style expansion (v0.3+)
3. Re-run eval on the same holdout; keep the change iff Recall@10 or MRR improves on holdout, or a bucket improves without hurting another by > 2%
4. Record: metric deltas, change description, date — appended to `docs/eval/tuning-log.md`
5. CI gate (once fixtures exist): retrieval-quality test asserts Recall@10 ≥ floor; PRs that regress it fail

Known levers ranked by expected value: (1) symbol-exact boost magnitude, (2) RRF k and list sizes, (3) chunk granularity for MD/config vs code, (4) identifier short-circuit threshold.

## 7. Privacy and data hygiene

- Session snapshots are temporary by design (§2.1). Nothing under `~/.local/share/opencode/` is copied into the repo.
- `eval.jsonl` derived from private repos (KanPad, taskpad-rs) stays **out of git** (`.gitignore`); it lives in `~/.local/share/codebase-intel/eval/` if needed at all.
- The OSS repo ships **fixtures generated against public repositories** (plan: taskpad-rs + one larger public Rust repo) by running agents there and mining *those* sessions; synthetic fallback if insufficient. Fixture generation script is part of F8/F9.
- No prompt text, file content from private repos, or session metadata is committed. CI runs a secret-pattern scan over `tests/fixtures/`.
- Mining never sends data anywhere — everything runs locally. Ollama is not involved in mining or evaluation (except when embedding chunks during fixture index builds).

## 8. Deliverables & exit criteria

| Deliverable | Where | Exit criterion |
|---|---|---|
| Miner CLI (`mine-sessions`) | `src/mine/` + `codebase-intel mine-sessions` | US-401 AC1–AC3 green |
| Eval harness (`eval`) | `src/eval/` | US-403 AC1–AC3 green; baseline report committed |
| Fixtures | `tests/fixtures/eval/*.jsonl` (public-repo derived) | reviewed for privacy; CI green |
| Tuning log + final weights | `docs/eval/` | US-404 AC1–AC3 green |
| CI retrieval gate | `.github/workflows` | blocks Recall@10 regression |

## 9. Open questions

| ID | Question | Leaning |
|---|---|---|
| OQ1 | Include `compaction` message text as extra task context? | No for v1 (noise); revisit if extraction quality is poor |
| OQ2 | Use `session_message.time_updated` for churn joins with git log? | Yes for v0.3 git intel joins (recency feature) |
| OQ3 | Cross-project eval (train on KanPad, test on taskpad-rs)? | Nice for generalization claims; decide after first baseline |
| OQ4 | Embed task texts for train/test dedup? | Yes, cosine ≥ 0.9 ⇒ drop — cheap and catches rephrasings |
