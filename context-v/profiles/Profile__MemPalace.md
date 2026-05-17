---
name: MemPalace Profile
slug: mempalace
upstream: https://github.com/MemPalace/mempalace
package: mempalace (PyPI), mempalace-mcp (CLI)
license: MIT
maintainer: MemPalace Contributors (milla-jovovich, @bensig)
study: studies/memory-layers-for-agents
profile_path: studies/memory-layers-for-agents/mempalace
profile_kind: library + mcp-server + cli
date_created: 2026-05-17
---

# MemPalace — Profile

A profile of MemPalace as it lives in this study (`studies/memory-layers-for-agents/mempalace/`). Cites pinned paths so you can jump to source rather than trust paraphrase. Read alongside [`Profile__Mem0.md`](./Profile__Mem0.md) — MemPalace is the closest direct peer in the study and explicitly defines itself in opposition to Mem0's design choices.

## TL;DR

MemPalace's bet is the simplest in the study and the most counterintuitive: **store the verbatim text, embed it with a default ONNX model, search it with hybrid semantic + BM25, and stop there**. No LLM-driven extraction step (the thing Mem0 spends a benchmark turn on). No graph DB for "graph memory." No supersession chain. Just text + embeddings + a thoughtful ranking layer + a deliberate refusal to throw anything away.

The reward for that simplicity is the benchmark wall (`benchmarks/BENCHMARKS.md:1-95`, `README.md:87-99`): **96.6% R@5 on LongMemEval with zero LLM calls**, scaling to **100% with optional Haiku rerank** (with the project being honest that the 100% involved tuning on 3 wrong answers — the held-out generalizable figure is **98.4%**). On **ConvoMem** (Salesforce 75K-QA), MemPalace reports **92.9% recall vs Mem0's 30-45%** — a roughly 2× margin attributed directly to Mem0's lossy extraction step (`benchmarks/BENCHMARKS.md:111`).

The storage spine is ChromaDB (default, pluggable per RFC 001) holding two collections — **mempalace_drawers** (verbatim chunks, 800-char default) and **mempalace_closets** (AAAK-compressed index pointing back to drawers) — plus a SQLite **temporal entity-relationship graph** for cross-fact navigation (`knowledge_graph.py:1-36`, `README.md:137-140`). The "palace" structure (Wings → Rooms → Drawers / Closets) is the user-facing taxonomy, inspired explicitly by Zettelkasten + Method of Loci (`MISSION.md:18`, `CLAUDE.md:29-34`).

Operationally: an MCP server (29 tools) is the primary surface for Claude Code / Cursor, with a Python library and a CLI for ingest, status, and repair. Local-first, no telemetry, no API key required for core memory. MIT-licensed, 52k+ stars.

**One sentence:** *MemPalace argues that the field has been over-engineering the extraction step, and proves it by storing verbatim text + hybrid-ranked semantic search and outscoring every extraction-based peer on the published benchmarks.*

## Why this exists — the design bet

Read `MISSION.md` end-to-end and the thesis is unmissable (`MISSION.md:8-22`): the author wanted agents to stop "waking up every day like 'what are we doing today'", looked at the field's RAG systems, decided they "felt like large empty warehouses," and reached back to Luhmann's Zettelkasten and the classical Method of Loci for the idea of *networked, not hierarchical, recall*.

That gets translated into four hard design moves:

1. **Verbatim text is the source of truth.** Nothing gets summarized into a memory record. Drawers are 800-char raw chunks; the embedding is computed on them; the embedding never replaces them (`mempalace/miner.py:79`, `CLAUDE.md:48`).
2. **Append-only and idempotent.** "Incremental only — Append-only ingest after initial build. Never destroy existing data to rebuild." (`CLAUDE.md:49-50`). File mtime + a schema-version field gate re-mining; deterministic IDs make re-ingest a no-op (`palace.py:49-57`, `miner.py:853`).
3. **Closets help, never gate.** A compressed AAAK index (closets) provides a ranking *signal*, never a filter, so even if compression mis-summarizes, the underlying drawer can still surface (`searcher.py:7-9`).
4. **Privacy by architecture.** No telemetry. No required cloud calls. Embeddings via on-device ONNX. The benchmark numbers don't require an API key.

If Mem0's bet is "extract intelligently and you'll win recall," MemPalace's counter-bet is "extract nothing and the embeddings will do the work, because lossy extraction is the silent killer of recall."

## The palace structure

(`CLAUDE.md:26-33`, `README.md:38-41`)

- **Wing** — broad category (a project, a person, a topic).
- **Room** — time-based (a day, a session) or topic-based grouping within a wing.
- **Drawer** — a verbatim chunk (default 800 chars) of source content.
- **Closet** — an AAAK-compressed index entry pointing at a set of drawer IDs.
- **Tunnel** — a cross-wing edge (`mempalace_create_tunnel` / `mempalace_find_tunnels`, MCP tools).

The naming is a deliberate UX choice. Where Mem0 says "memory" and Volt says "summary node," MemPalace says "drawer" — which makes the verbatim-storage semantics legible even to a non-engineer. The Method-of-Loci framing carries through.

## Storage topology

Three layers:

| Layer | Backend | Purpose | Code |
|---|---|---|---|
| Primary | ChromaDB (pluggable per RFC 001) | `mempalace_drawers` (verbatim text + ONNX embeddings) and `mempalace_closets` (AAAK index) | `mempalace/backends/base.py:1-14`, `README.md:38-41` |
| Graph | SQLite (local, no external dep) | Temporal entity-relationship graph | `knowledge_graph.py:1-36`, `README.md:137-140` |
| Files | Filesystem | `.mempalace/` palace dir + per-project `entities.json` and `mempalace.yaml` | `mempalace/config.py` |

The roadmap signals **swappable backends** (`ROADMAP.md:29-36`): PostgreSQL (ACID/concurrent), LanceDB (multi-device sync), and a bespoke PalaceStore — all hidden behind the RFC 001 abstraction so the rest of the system doesn't change.

Notable for what's **not** there: no Neo4j (the graph is SQLite); no separate vector DB tier (Chroma's HNSW is doing the work); no LLM-summarized memory store (drawers are raw). The graph is a *navigation* layer, not a *storage* layer.

## Schema of a drawer record

ChromaDB collection entry (`miner.py:768-840, 853, 938`):

```python
# documents[]
"<raw chunk, up to 800 chars verbatim>"

# ids[]
"drawer_{wing}_{room}_{sha256_of_(source_file, chunk_index)[:24]}"

# metadata
{
  "wing": "...",                # project / person
  "room": "...",                # session / topic
  "source_file": "...",
  "chunk_index": int,
  "added_by": "mcp" | "claude-code" | ...,
  "filed_at": "ISO 8601 UTC",
  "source_mtime": float,        # for idempotent re-mine
  "hall": "code|notes|research|...",   # detected category
  "entities": "Alice;Bob;...",  # max 25, semicolon-joined
  "normalize_version": int,     # schema versioning for rebuild eligibility
  "ingest_mode": "registry|project|convo|..."
}
```

Notice what's **missing** vs the rest of the study:

- **No confidence field** — every drawer is equal-weight once indexed.
- **No supersession pointers** — corrections don't invalidate prior drawers; they get filed as additional drawers and ranked alongside.
- **No explicit embedding reference** — Chroma owns the embedding; the drawer is the text.
- **No provenance chain** — `added_by` + `filed_at` is the whole audit trail.

This is the same flatness Mem0 chose (`Profile__Mem0.md` §schema) but combined with verbatim retention rather than LLM extraction — which is what makes the recall numbers move.

## Write policy — append-only, idempotent, never destructive

The whole policy fits on a card:

> Incremental only — Append-only ingest after initial build. Never destroy existing data to rebuild. (`CLAUDE.md:49-50`)

Concretely (`miner.py:853, 938`, `palace.py:49-57`):

- IDs are deterministic SHA-256 of `(source_file, chunk_index)` — re-ingesting the same content is a no-op.
- `file_already_mined()` checks `source_mtime` + `normalize_version` — only stale or missing chunks are re-read.
- On re-mine of a changed file, `collection.delete(where={"source_file": source_file})` clears the prior chunks **for that file** before upserting the fresh ones. Other files in the same wing are untouched.
- **No LLM call in the write path.** No extraction prompt. No summarization. This is the budget-defining design choice — writes are essentially "chunk + embed + upsert."

Background filing happens through Claude Code hooks (`hooks/mempal_save_hook.sh`, `mempal_precompact_hook.sh`, see `MISSION.md:24-30`) — the Stop and PreCompact hooks fire idempotent CLI commands so the agent never blocks waiting on a memory write.

## Recall API — hybrid semantic + BM25 + closet-boost

Primary surface (`mcp_server.py:855-925`, `searcher.py:748+`):

```python
search_memories(
    query: str,
    palace_path: str,
    wing: Optional[str] = None,
    room: Optional[str] = None,
    n_results: int = 5,
)
```

Ranking (`searcher.py:63-193`):

1. **Vector retrieval** via ChromaDB ONNX embedding → candidate set.
2. **Okapi BM25** scored over the candidate set (`searcher.py:63-120`).
3. **Closet boost** — if the compressed index (closet) agrees with a drawer, apply rank boost (`searcher.py:180-193`).
4. **Optional `where`-clause filter** — scope by wing / room / metadata predicate.

Two things to notice:

- **Raw mode is production-cost-free.** No reranker, no LLM call at query time. That's how the 96.6% LongMemEval R@5 number is reached with zero API calls.
- **Closets-as-signal, never gate** (`searcher.py:7-9`): "Closets are a ranking signal, never a gate, so weak closets (regex extraction on narrative content) can only help, never hide drawers." This is a structural safety: lossy compression never costs you a drawer it should have surfaced.

**Optional LLM rerank** (separate from `search_memories`) re-orders the top-20 candidates using Haiku or Sonnet (`README.md:100-106`). That's the path to the 100% LongMemEval number.

The **hybrid v5** path adds keyword boosting on entity names / quoted phrases / temporal markers, temporal-proximity boosting, and preference-pattern extraction (`ROADMAP.md:49-52`).

## Eviction & compaction — none, by design

**There is no TTL, no eviction, no LLM-driven compaction.** Retention is infinite by design (`CLAUDE.md:51-52`).

What exists instead:

- **HNSW index bloat prevention** (v3.1, `backends/chroma.py:107-130`, `ROADMAP.md:22`): tuned batch + sync thresholds collapse a 441 GB `link_lists.bin` down to 433 KB on large palaces. This is an **index-structure** fix, not a data-deletion policy.
- **`mempalace compress`** CLI command — metadata rebuild, not trimming.
- **`mempalace repair`** — auto-detects HNSW structural corruption and quarantines broken segments.

This is the most "store everything forever" stance in the study. Compare to Volt (Upward: never evict; Dolt: evict bindles into lineage pointers), Neo (multiple coordinated eviction mechanisms), Mem0 (none in OSS but schema for it in `openmemory`). MemPalace simply commits.

## Scopes & namespacing

Four-level (`README.md:149-154`, `mempalace/mcp_server.py:1503-1580`):

| Scope | Purpose |
|---|---|
| Agent / Persona | Each agent gets its own wing + per-agent **diary** (read/write via `mempalace_diary_*` MCP tools) |
| Wing | Top-level (project / person), discoverable via `mempalace_list_agents` |
| Room | Subdivides a wing (a day, a session, a named topic) |
| Drawer | Individual verbatim chunk |

**Multi-user / tenant isolation** is handled by separate palace paths — there is no built-in RBAC. The trust boundary is the user UID + filesystem permissions on `.mempalace/`. This is consistent with the local-first stance; if you need multi-tenant SaaS, you're wiring it yourself.

## Operational story — MCP-server-first

Three surfaces (`README.md:142-163`):

1. **MCP server** (primary). `mempalace-mcp [--palace /path/]`. Exposes **29 tools** spanning palace reads/writes, knowledge-graph ops (`mempalace_kg_query`, `mempalace_kg_add`, `mempalace_kg_invalidate`, `mempalace_kg_timeline`), cross-wing tunnels (`mempalace_traverse_graph`, `mempalace_create_tunnel`, `mempalace_follow_tunnels`), and per-agent diaries. Tool dict at `mcp_server.py:1819+`.
2. **CLI**. `mempalace init <dir>` (auto-detect rooms from folder structure), `mempalace mine <dir>` (ingest files), `mempalace mine <dir> --mode convos` (ingest Claude Code / ChatGPT transcripts), `mempalace search "<query>"`, `mempalace wake-up [--wing X]` (returns L0+L1 context for a fresh session), `mempalace status`, `mempalace repair`.
3. **Python library**. `from mempalace import Palace, search_memories`. Composable **layers** (`layers.py:1-80`): Layer0 (identity), Layer1 (essential story), Layer2 (on-demand), Layer3 (deep search). Direct ChromaDB access via `mempalace.backends`.

**Claude Code hooks** (`hooks/`, `mcp_server.py:1-21`): Stop and PreCompact hooks fire idempotent `mempalace.hooks_cli` commands in the background so chat transcripts get filed without polluting the chat window. `MISSION.md:24-34` records this as a v4 redesign that cut ~$1.13/session in token cost by no longer re-transmitting diary writes inline.

## The benchmark wall — and the honesty on it

(`README.md:87-99`, `benchmarks/BENCHMARKS.md:1-115`)

| Benchmark | Raw (zero LLM) | With rerank | Notes |
|---|---|---|---|
| **LongMemEval** (500q) | **96.6% R@5** | 100% (Haiku rerank) | 100% involved teaching-to-test on 3 answers; held-out 450q figure: **98.4%** |
| **LoCoMo** (1,986 multi-hop) | 88.9% R@10 (hybrid v5) | 100% R@5/R@10 (Sonnet) | |
| **ConvoMem** (Salesforce, 75K QA) | **92.9%** | — | Mem0 reports 30–45% on same. ~2× advantage. |
| **MemBench** (ACL 2025, 8.5K) | **80.3% R@5** | — | |

What's worth flagging — both for and against:

- **For:** The headline 96.6% is reproducible with `benchmarks/longmemeval_bench.py` and requires no API key. The methodology is documented (`benchmarks/BENCHMARKS.md:1-40`), including the framing finding: *"Raw chromadb with default embeddings scores 96.6%. Nobody published this because nobody tried it and measured properly."*
- **For (honesty):** `benchmarks/BENCHMARKS.md:70-95` explicitly walks through the test-set tuning that produces 100% and reports 98.4% as the generalizable number. That kind of self-correction is rare in the space.
- **Against:** Cross-system comparisons are not apples-to-apples — `README.md:117-121` is upfront that Mem0 / Mastra / Supermemory publish on different metrics and different splits, so the published "MemPalace vs X" tables in marketing are weaker than the methodology section makes the benchmark itself.
- **Worth running independently:** StateBench (`Profile__StateBench.md`) would test MemPalace on a different axis entirely — Superseded Fact Resurrection Rate. The structural prediction: MemPalace's "never throw anything away" stance is exposed on SFRR (it will happily surface a superseded fact alongside the correction). Decision accuracy and must-mention should be strong; SFRR is the open question.

## What's inside this submodule

| Path | What's there |
|---|---|
| `README.md` | Pitch, benchmarks, install, MCP tool list pointer |
| `MISSION.md` | The "why" — Zettelkasten + Method of Loci framing, v4 design rationale |
| `CLAUDE.md` | Agent-facing notes (palace structure, write discipline) |
| `ROADMAP.md` | Backend pluggability (PostgreSQL, LanceDB, PalaceStore), hybrid v5, defrag |
| `CHANGELOG.md` | Version history |
| `mempalace/` | Python package — `miner.py`, `searcher.py`, `palace.py`, `layers.py`, `knowledge_graph.py`, `config.py`, `repair.py`, `mcp_server.py`, `hooks_cli.py`, `backends/` |
| `mempalace/backends/base.py` | RFC 001 backend abstraction |
| `benchmarks/` | LongMemEval / LoCoMo / ConvoMem / MemBench harnesses + `BENCHMARKS.md` |
| `hooks/` | Claude Code Stop / PreCompact shell hooks |
| `integrations/`, `tools/` | Extensibility surface |
| `examples/`, `docs/` | Worked examples + reference docs |
| `landing/`, `website/` | Marketing site source |
| `tests/` | pytest suite |
| `openarena-claim.txt` | OpenArena ownership verification token |

If you read three files: `MISSION.md` (the why), `benchmarks/BENCHMARKS.md` (the proof + the honesty), `mempalace/searcher.py` (where the ranking actually happens).

## Mental model for using it well

- **Trust the verbatim default.** Don't try to "summarize before filing" — that's the move MemPalace is designed to make unnecessary.
- **Set up wings deliberately.** Wings are the coarsest filter; misclassified wings hurt recall more than over-chunked drawers.
- **Use the diary tools for agent-specific state.** Each agent's diary is its own namespace; this is how you avoid cross-agent memory contamination.
- **Add tunnels for cross-wing connections.** The graph is the navigation layer. If two wings are about related projects, a tunnel makes traversal cheap.
- **Run the hooks.** Background filing is what makes the "agent doesn't ask 'what are we doing today'" promise real. Without hooks you're filing manually.
- **Reach for rerank only when raw isn't enough.** Raw mode is free; rerank costs LLM calls. The benchmarks say raw is enough for most cases.
- **Back up `.mempalace/`.** No built-in sync. Until LanceDB lands (`ROADMAP.md:59`), this is the user's job.

## When NOT to reach for this

- **Supersession matters.** If your domain needs "this fact replaced that fact," MemPalace will resurrect the old one. Reach for Neo (`Profile__Neo.md`) or a state-based engine (see Memgine in `Profile__StateBench.md`).
- **You need per-user-scope cross-session memory at multi-tenant scale.** MemPalace has wings, not tenants. Bolt scope on, or pick Mem0 which makes user scope mandatory.
- **You need cloud sync today.** Local-first by design. The LanceDB roadmap entry is the answer here, but it isn't shipped yet.
- **You want LLM-driven extraction.** That's the *not*-MemPalace pitch. The whole project argues you don't.
- **Markdown + git is enough.** As with the other entries: `context-vigilance` is a meaningfully simpler answer when the consumer is a human and the structure is the value.

## How this compares to the rest of the study

| Axis | MemPalace | Mem0 | Neo | Volt |
|---|---|---|---|---|
| **Write** | Verbatim, append-only, idempotent | LLM extraction (single-pass v3) | Append + deterministic supersession (cosine 0.85) | Verbatim immutable log + async LLM summary |
| **Recall** | Hybrid (vector + BM25 + closet-boost), zero-LLM default | Hybrid (vector + BM25 + entity-link), single-LLM extract on write | Semantic × confidence × success | DAG of summary nodes; raw still retrievable |
| **Supersession** | None (and explicitly chosen) | None in OSS | First-class | Implicit in time-ordered log |
| **Eviction** | None | None in OSS | Multiple coordinated mechanisms | Dolt: evict bindles; Upward: never |
| **Storage** | Chroma (drawers + closets) + SQLite graph | Vector + entity store + SQLite | Scoped JSON files | Postgres DAG |
| **Scope axis** | Wing / Room | user_id / agent_id / run_id (required) | global / org / project / session (auto) | conversation / session |
| **Benchmark posture** | "Best-benchmarked" — published with methodology, including honesty about test-tuning | Strong on LoCoMo / LongMemEval / BEAM | Outcome detection on real codebases | OOLONG long-context claim |
| **Operational** | MCP + CLI + library, local-first | Library / server / hosted | CLI + library + Claude/Codex plugins | Embedded coding agent |
| **Closest comparison** | Designed in opposition to Mem0 | Designed for general agent recall | Designed for code-reasoning loops | Designed for long-horizon coding |

The MemPalace-vs-Mem0 axis is the most informative comparison in the study now. Same problem (give agents persistent memory), same general toolchain (vector store + Python), nearly opposite bets at the write step. The benchmarks suggest the verbatim bet wins on retrieval recall by a meaningful margin. The unanswered question is whether the verbatim bet *loses* on state-correctness-over-time — exactly what StateBench would measure.

## How this compares to our own `context-vigilance` skill

MemPalace and `context-vigilance` share the "don't throw anything away" instinct and the local-first posture, but they're aimed at different consumers:

| | MemPalace | context-vigilance |
|---|---|---|
| **Consumer** | Agent (via MCP) | Human + agent |
| **Storage** | Chroma chunks + SQLite graph | Markdown files in git |
| **Granularity** | 800-char drawers | Section-level markdown |
| **Recall** | Hybrid vector + BM25, ranked | Human reading + grep |
| **Authoring** | Hooks file automatically | Humans edit, PRs review |
| **Compression** | AAAK closets (lossy index, lossless drawers) | Versioning + frontmatter |
| **Audit** | `filed_at` + `added_by` + immutable drawer | git log |

They compose cleanly: `context-vigilance` for the durable, reviewed project knowledge; MemPalace for the long tail of session transcripts, tool outputs, and verbatim conversation history that the agent should be able to fuzzy-recall without a human filing it. In a Lossless setup you could plausibly point a MemPalace wing at a `context-v/` directory and let agents query both surfaces — the markdown for structure, the palace for everything else.

The transferable lesson from MemPalace back to `context-vigilance`: **the AAAK closet-as-signal-never-gate pattern**. If we ever build a fuzzy index over `context-v/`, treat any summary layer as a ranking hint, not a filter. The underlying file should always be retrievable.

## One-line summary

> MemPalace argues — and benchmarks — that the field's extraction-and-summarize memory pattern is over-engineered; that storing verbatim chunks in ChromaDB with hybrid semantic + BM25 ranking + an AAAK closet-as-signal layer outperforms the extraction systems on LongMemEval / LoCoMo / ConvoMem / MemBench while costing zero LLM calls at query time; and that "store everything, never lose information, rank cleverly" is a stronger baseline than anyone was measuring.
