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
site_uuid: 9aae1b02-1ea8-4795-9a4a-4527e8fe1f5e
hex_code: lgvblv
date_authored_initial_draft: 2026-05-17
date_authored_current_draft: 2026-11-23
lede: >-
  The counter-bet: extract nothing. Raw 800-character chunks score 96.6% R@5
  on LongMemEval at zero LLM calls.
summary: >-
  Profile of MemPalace as pinned in the memory-layers-for-agents study. It is
  the direct structural opposite of Mem0 — same problem, same toolchain, nearly
  inverted bet at the write step — and the pairing is the study's most
  informative. Covers the Wing/Room/Drawer/Closet/Tunnel taxonomy and its
  Zettelkasten and Method-of-Loci lineage, the ChromaDB drawers-plus-closets
  layout with a SQLite temporal knowledge graph beside it, the append-only
  idempotent write policy with deterministic SHA-256 IDs, hybrid vector + BM25
  recall with the closets-as-signal-never-a-gate rule, the deliberate absence of
  eviction, the MCP/CLI/library surfaces and Claude Code hooks, and an honest
  reading of the benchmark table including its own caveats. The transferable
  lesson is the closet-as-signal pattern: any summary layer over a corpus should
  be a ranking hint, never a filter.
publish: true
title: MemPalace — Profile
date_modified: 2026-11-23
authors:
  - Michael Staton
status: Reference
type: Profiles
---

# MemPalace — Profile

A profile of MemPalace as it lives in this study (`studies/memory-layers-for-agents/mempalace/`). Cites pinned paths so you can jump to source rather than trust paraphrase. Read alongside [`Profile__Mem0.md`](./Profile__Mem0.md) — MemPalace is the closest direct peer in the study and explicitly defines itself in opposition to Mem0's design choices.

## TL;DR

MemPalace's bet is the simplest in the study and the most counterintuitive: **store the verbatim text, embed it with a default ONNX model, search it with hybrid semantic + BM25, and stop there**. No LLM-driven extraction step (the thing Mem0 spends a benchmark turn on). No graph DB for "graph memory." No supersession chain on drawers (the side knowledge graph gained an atomic `supersede()` in v3.6.0, but drawers themselves are never superseded). Just text + embeddings + a thoughtful ranking layer + a deliberate refusal to throw anything away.

The reward for that simplicity is the benchmark wall (`benchmarks/BENCHMARKS.md`, `README.md:241-269`): **96.6% R@5 on LongMemEval with zero LLM calls**, vendor-reported, and **≥99% with an optional LLM rerank** (Haiku, Sonnet, or an open model). An earlier README headlined 100%; the current one declines to, because the last 0.6% came from inspecting specific wrong answers, and it names the held-out **98.4%** as the generalizable figure (`README.md:246-260`). On **ConvoMem** (Salesforce 75K-QA), MemPalace reports **92.9% recall vs Mem0's 30-45%** — a roughly 2× margin attributed directly to Mem0's lossy extraction step (`benchmarks/BENCHMARKS.md:111`).

The storage spine is ChromaDB (default, pluggable per RFC 001) holding two collections — **mempalace_drawers** (verbatim chunks, 800-char default) and **mempalace_closets** (a regex-built pointer index, `topic|entities|→drawer_ids`, pointing back to drawers; AAAK compression is a separate opt-in via `mempalace compress`) — plus a SQLite **temporal entity-relationship graph** for cross-fact navigation (`mempalace/knowledge_graph.py`, `CLAUDE.md:108-125`). The "palace" structure (Wings → Rooms → Drawers / Closets) is the user-facing taxonomy, inspired explicitly by Zettelkasten + Method of Loci (`MISSION.md`, `CLAUDE.md:11-17`). Between v3.6 and v3.10 the backend became a real choice: `chroma` (default), `sqlite_exact`, `rust_exact`, `milvus`, `qdrant`, `pgvector` (`README.md:189-206`).

Operationally: an MCP server (47 tools as of v3.12, `README.md:296-302`) is the primary surface for Claude Code, Cursor, Codex, and Antigravity, with a 3-tool lightweight server (`mempalace/mcp_light_server.py`), an HTTP hub (`mempalace serve`), a Python library, and a CLI for ingest, status, and repair. Local-first, no telemetry, no API key required for core memory. MIT-licensed; 52k+ stars at the May 2026 pin (not re-verified).

**One sentence:** *MemPalace argues that the field has been over-engineering the extraction step, and proves it by storing verbatim text + hybrid-ranked semantic search and outscoring every extraction-based peer on the published benchmarks.*

## Why this exists — the design bet

Read `MISSION.md` end-to-end and the thesis is unmissable (`MISSION.md:8-22`): the author wanted agents to stop "waking up every day like 'what are we doing today'", looked at the field's RAG systems, decided they "felt like large empty warehouses," and reached back to Luhmann's Zettelkasten and the classical Method of Loci for the idea of *networked, not hierarchical, recall*.

That gets translated into four hard design moves:

1. **Verbatim text is the source of truth.** Nothing gets summarized into a memory record. Drawers are 800-char raw chunks; the embedding is computed on them; the embedding never replaces them (`DEFAULT_CHUNK_SIZE = 800` at `mempalace/config.py:415`, `CLAUDE.md:23`).
2. **Append-only and idempotent.** "Incremental only — Append-only ingest after initial build. Never destroy existing data to rebuild. A crash mid-operation must leave the existing palace untouched." (`CLAUDE.md:24`). File mtime + a schema-version field gate re-mining; deterministic IDs make re-ingest a no-op (`mempalace/palace/mined.py:161`, `mempalace/ids.py:55-67`).
3. **Closets help, never gate.** The closet index provides a ranking *signal*, never a filter, so even if it mis-summarizes, the underlying drawer can still surface (`mempalace/searcher/__init__.py:7`, `mempalace/searcher/query.py:120`).
4. **Privacy by architecture.** No telemetry. No required cloud calls. Embeddings via on-device ONNX. The benchmark numbers don't require an API key.

If Mem0's bet is "extract intelligently and you'll win recall," MemPalace's counter-bet is "extract nothing and the embeddings will do the work, because lossy extraction is the silent killer of recall."

## The palace structure

(`CLAUDE.md:11-17`, `CLAUDE.md:108-125`)

- **Wing** — broad category (a project, a person, a topic).
- **Room** — time-based (a day, a session) or topic-based grouping within a wing.
- **Drawer** — a verbatim chunk (default 800 chars) of source content.
- **Closet** — a regex-built index entry (pointer lines up to ~1,500 chars, `mempalace/palace/closets.py`) pointing at a set of drawer IDs.
- **Hallway** — a within-wing edge between entities that co-occur in a drawer, derived automatically since v3.3.6 and stored in `hallways.sqlite3` since v3.12 (`mempalace/hallways.py`).
- **Tunnel** — a cross-wing edge (`mempalace_create_tunnel` / `mempalace_find_tunnels`, MCP tools).

The naming is a deliberate UX choice. Where Mem0 says "memory" and Volt says "summary node," MemPalace says "drawer" — which makes the verbatim-storage semantics legible even to a non-engineer. The Method-of-Loci framing carries through.

## Storage topology

Three layers:

| Layer | Backend | Purpose | Code |
|---|---|---|---|
| Primary | ChromaDB default; `sqlite_exact`, `rust_exact`, `milvus`, `qdrant`, `pgvector` opt-in | `mempalace_drawers` (verbatim text + ONNX embeddings) and `mempalace_closets` (pointer index) | `mempalace/backends/base.py`, `mempalace/backends/`, `README.md:189-206` |
| Graph | SQLite (local, no external dep) | Temporal entity-relationship graph | `mempalace/knowledge_graph.py`, `README.md:288-293` |
| Files | Filesystem | `.mempalace/` palace dir + per-project `entities.json` and `mempalace.yaml` | `mempalace/config.py` |

The roadmap's **swappable backends** promise (`ROADMAP.md:29-36`, still the May text) has mostly shipped, in a different shape: pgvector, Qdrant, and Milvus as server options, `sqlite_exact` as an exact NumPy scan, and `rust_exact` as a native Rust engine that reads the same SQLite file (`README.md:189-210`, `crates/`). LanceDB and PalaceStore have not landed.

Notable for what's **not** there: no Neo4j (the graph is SQLite); no separate vector DB tier by default (Chroma's HNSW is doing the work unless you opt into another backend); no LLM-summarized memory store (drawers are raw). The graph is a *navigation* layer, not a *storage* layer.

## Schema of a drawer record

ChromaDB collection entry (`mempalace/miner.py`; ID helpers in `mempalace/ids.py:55-130`). Since v3.12, `filed_at` is always UTC with an explicit offset, and conversation drawers also carry `authored_at` (transcript time, added in v3.6.0):

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

> Incremental only — Append-only ingest after initial build. Never destroy existing data to rebuild. A crash mid-operation must leave the existing palace untouched. (`CLAUDE.md:24`)

Concretely (`mempalace/ids.py`, `mempalace/palace/mined.py:161`, `mempalace/retention.py`, `mempalace/revisions.py`):

- IDs are deterministic SHA-256 of `(source_file, chunk_index)` for project files (`ids.py:55`), of `(source_file, extract_mode, chunk_index)` for conversations, and of `(wing, room, content)` for MCP `add_drawer` — re-ingesting the same content is a no-op.
- `file_already_mined()` checks `source_mtime` + `normalize_version` — only stale or missing chunks are re-read. v3.12 bumped the normalization version to 3, so every source is re-read once; unchanged drawers keep their stored embeddings.
- On re-mine of a changed file, the miner writes the new drawers and then prunes the old ones **for that file**, and since the post-3.12 develop branch it prunes only drawers its own producer filed. `retention.py` splits drawers into "preserve" (conversations, diaries, manual entries, where the palace is the only copy) and "index current" (project files, which may be replaced). Before a conversation or diary re-mine overwrites a drawer, its old text is appended to `drawer_revisions.jsonl` (`revisions.py:24`), so a compacted transcript no longer erases what was mined from the longer version. That journal is not searchable yet.
- **No LLM call in the write path.** No extraction prompt. No summarization. This is the budget-defining design choice — writes are essentially "chunk + embed + upsert."

Background filing happens through Claude Code hooks (`hooks/mempal_save_hook.sh`, `hooks/mempal_precompact_hook.sh`, now thin wrappers over `mempalace/hooks_cli.py`; see `MISSION.md`), with matching hooks for Cursor, Codex, and Antigravity since v3.4 — the Stop and PreCompact hooks fire idempotent CLI commands so the agent never blocks waiting on a memory write.

## Recall API — hybrid semantic + BM25 + closet-boost

Primary surface (`mempalace/searcher/query.py:277`; the MCP wrapper is in `mempalace/mcp_server/tools_read.py`). The signature below is the May shape; the current function also takes `since`/`before` date bounds (v3.7), a `source_file` filter (v3.5), and a `candidate_strategy` whose `"lexical"` value runs BM25 only without loading the embedding model (post-3.12 develop):

```python
search_memories(
    query: str,
    palace_path: str,
    wing: Optional[str] = None,
    room: Optional[str] = None,
    n_results: int = 5,
)
```

Ranking (the former `searcher.py` is now the `mempalace/searcher/` package: `candidates.py`, `ranking.py`, `query.py`, `sqlite_bm25.py`):

1. **Vector retrieval** via the backend's embedding → candidate set (`searcher/candidates.py`).
2. **Okapi BM25** scored over the candidate set (`mempalace/lexical.py:26`).
3. **Closet boost** — if the closet index agrees with a drawer, apply a rank boost through `effective_distance` (`searcher/ranking.py:189-212`). Since v3.9 the public `similarity` field reports the raw vector score, so the boost shows in rank order only.
4. **Optional `where`-clause filter** — scope by wing / room / metadata predicate.

Two things to notice:

- **Raw mode is production-cost-free.** No reranker, no LLM call at query time. That's how the 96.6% LongMemEval R@5 number is reached with zero API calls.
- **Closets-as-signal, never gate** (`searcher/__init__.py:7`, `searcher/query.py:436`): "Closets are a ranking signal, never a gate, so weak closets (regex extraction on narrative content) can only help, never hide drawers." This is a structural safety: lossy compression never costs you a drawer it should have surfaced.

**Optional LLM rerank** (separate from `search_memories`) re-orders the top-20 candidates with any capable model; the README reports Haiku, Sonnet, and minimax-m2.7 (`README.md:255-260`). It is the path to the ≥99% LongMemEval figure.

The **hybrid v5** path adds keyword boosting on entity names / quoted phrases / temporal markers, temporal-proximity boosting, and preference-pattern extraction (`ROADMAP.md:49-52`).

## Eviction & compaction — none, by design

**There is no TTL, no eviction, no LLM-driven compaction.** Retention is infinite by design (`CLAUDE.md:23-24`). v3.12 went further and removed `python -m mempalace.dedup`, because it could delete real memories under 20 characters.

What exists instead:

- **HNSW index bloat prevention** (v3.1, `backends/chroma.py:116-190`, `ROADMAP.md`): tuned batch + sync thresholds collapse a 441 GB `link_lists.bin` down to 433 KB on large palaces. This is an **index-structure** fix, not a data-deletion policy.
- **Decay on navigation edges only** (v3.3.6, `mempalace/dynamics.py`): hallways and tunnels strengthen when reinforced and fade when a connection stops recurring. Drawers never decay.
- **`mempalace compress`** CLI command — metadata rebuild, not trimming.
- **`mempalace repair`** — auto-detects HNSW structural corruption and quarantines broken segments.

This is the most "store everything forever" stance in the study. Compare to Volt (Upward: never evict; Dolt: evict bindles into lineage pointers), Neo (multiple coordinated eviction mechanisms), Mem0 (none in OSS but schema for it in `openmemory`). MemPalace simply commits.

## Scopes & namespacing

Four-level (`README.md`, `mempalace/mcp_server/tools_diary.py`). Since v3.12 every producer (CLI, miners, diary, and the Claude Code, Codex, Cursor, and Antigravity hooks) derives a project's wing from one slug rule, so a session's diary and transcript land in the same wing:

| Scope | Purpose |
|---|---|
| Agent / Persona | Each agent gets its own wing + per-agent **diary** (read/write via `mempalace_diary_*` MCP tools) |
| Wing | Top-level (project / person), discoverable via `mempalace_list_agents` |
| Room | Subdivides a wing (a day, a session, a named topic) |
| Drawer | Individual verbatim chunk |

**Multi-user / tenant isolation** is handled by separate palace paths — there is no built-in RBAC. The trust boundary is the user UID + filesystem permissions on the palace directory (`~/.config/mempalace` for new installs since v3.10; older installs keep `~/.mempalace`). The newer server backends (Milvus, Qdrant, pgvector) add namespaces, and `mempalace serve` (v3.6) puts a bearer token in front of any non-loopback bind, but that is a shared-team server, not tenant isolation.

## Operational story — MCP-server-first

Three surfaces (`README.md`):

1. **MCP server** (primary). `mempalace-mcp [--palace /path/]`. Exposes **47 tools** at v3.12 (29 at the May pin) spanning palace reads/writes, knowledge-graph ops (`mempalace_kg_query`, `mempalace_kg_add`, `mempalace_kg_invalidate`, `mempalace_kg_supersede`, `mempalace_kg_timeline`), cross-wing tunnels (`mempalace_traverse_graph`, `mempalace_create_tunnel`, `mempalace_follow_tunnels`), per-agent diaries, and agent coordination (logstream events, tasks, artifacts). The former `mcp_server.py` is now the `mempalace/mcp_server/` package (`tools_read.py`, `tools_write.py`, `tools_kg.py`, `tools_diary.py`, `tools_coord.py`, `http.py`). A lightweight 3-tool server (`palace_query`, `palace_exec`, `palace_coordinate`, driven by a Palace Query Language) sits at `mempalace/mcp_light_server.py`, and `mempalace serve` runs an HTTP hub that serves both catalogs (`/mcp` and `/mcp/light`) to every agent on a machine.
2. **CLI**. `mempalace init <dir>` (auto-detect rooms from folder structure), `mempalace mine <dir>` (ingest files), `mempalace mine <dir> --mode convos` (ingest Claude Code / ChatGPT transcripts), `mempalace search "<query>"`, `mempalace wake-up [--wing X]` (returns L0+L1 context for a fresh session), `mempalace status`, `mempalace repair`.
3. **Python library**. `from mempalace import Palace, search_memories`. Composable **layers** (`layers.py:315-548`): Layer0 (identity), Layer1 (essential story), Layer2 (on-demand), Layer3 (deep search). Direct backend access via `mempalace.backends`.

**Claude Code hooks** (`hooks/`, `mempalace/hooks_cli.py`): Stop and PreCompact hooks fire idempotent `mempalace.hooks_cli` commands in the background so chat transcripts get filed without polluting the chat window. `MISSION.md:24-34` records this as a v4 redesign that cut ~$1.13/session in token cost by no longer re-transmitting diary writes inline.

## The benchmark wall — and the honesty on it

(`README.md:241-269`, `benchmarks/BENCHMARKS.md`; all figures vendor-reported)

| Benchmark | Raw (zero LLM) | With rerank | Notes |
|---|---|---|---|
| **LongMemEval** (500q) | **96.6% R@5** | ≥99% (any capable model) | The May README headlined 100%; the current one drops that number as teaching-to-test. Held-out 450q figure: **98.4%** |
| **LoCoMo** (1,986 multi-hop) | 88.9% R@10 (hybrid v5) | 100% R@5/R@10 (Sonnet) | |
| **ConvoMem** (Salesforce, 75K QA) | **92.9%** | — | Mem0 reports 30–45% on same. ~2× advantage. |
| **MemBench** (ACL 2025, 8.5K) | **80.3% R@5** | — | |

What's worth flagging — both for and against:

- **For:** The headline 96.6% is reproducible with `benchmarks/longmemeval_bench.py` and requires no API key. The methodology is documented (`benchmarks/BENCHMARKS.md:1-40`), including the framing finding: *"Nobody published this result because nobody tried the simple thing and measured it properly."* (`BENCHMARKS.md:20`)
- **For (honesty):** `benchmarks/BENCHMARKS.md:72-95` explicitly walks through the test-set tuning that produces 100% and reports 98.4% as the generalizable number. Between pins the README went one step further and stopped headlining 100% at all (`README.md:255-260`), though `BENCHMARKS.md:31-36` still presents it as "the competitive story". That kind of self-correction is rare in the space.
- **Against:** Cross-system comparisons are not apples-to-apples — `README.md:270-275` is upfront that Mem0 / Mastra / Supermemory publish on different metrics and different splits, so the published "MemPalace vs X" tables in marketing are weaker than the methodology section makes the benchmark itself.
- **Worth running independently:** StateBench (`Profile__StateBench.md`) would test MemPalace on a different axis entirely — Superseded Fact Resurrection Rate. The structural prediction: MemPalace's "never throw anything away" stance is exposed on SFRR (it will happily surface a superseded fact alongside the correction). Decision accuracy and must-mention should be strong; SFRR is the open question.

## What's inside this submodule

| Path | What's there |
|---|---|
| `README.md` | Pitch, benchmarks, install, MCP tool list pointer |
| `MISSION.md` | The "why" — Zettelkasten + Method of Loci framing, v4 design rationale |
| `CLAUDE.md` | Agent-facing notes (palace structure, write discipline) |
| `ROADMAP.md` | Backend pluggability (PostgreSQL, LanceDB, PalaceStore), hybrid v5, defrag. Not updated since the May pin; the CHANGELOG is the better guide to what shipped |
| `CHANGELOG.md` | Version history |
| `mempalace/` | Python package — `miner.py`, `convo_miner.py`, `searcher/`, `palace/`, `ids.py`, `retention.py`, `revisions.py`, `layers.py`, `knowledge_graph.py`, `hallways.py`, `dynamics.py`, `logstream.py`, `config.py`, `repair.py`, `mcp_server/`, `mcp_light_server.py`, `hooks_cli.py`, `backends/` |
| `mempalace/backends/base.py` | RFC 001 backend abstraction; siblings `chroma.py`, `sqlite_exact.py`, `rust_exact.py`, `milvus.py`, `qdrant.py`, `pgvector.py` |
| `crates/` | Rust workspace behind `rust_exact` and the `mempalace-native` CLI (new since the May pin) |
| `skills/`, `plugins/`, `rules/` | Agent setup skill (`npx skills add MemPalace/mempalace`), harness plugins, shared-brain rules (new since the May pin) |
| `benchmarks/` | LongMemEval / LoCoMo / ConvoMem / MemBench harnesses + `BENCHMARKS.md` |
| `hooks/` | Claude Code Stop / PreCompact shell hooks, plus Cursor and Antigravity hook sets |
| `integrations/`, `tools/` | Extensibility surface |
| `examples/`, `docs/` | Worked examples + reference docs |
| `landing/`, `website/` | Marketing site source |
| `tests/` | pytest suite |
| `openarena-claim.txt` | OpenArena ownership verification token |

If you read three files: `MISSION.md` (the why), `benchmarks/BENCHMARKS.md` (the proof + the honesty), `mempalace/searcher/query.py` and `searcher/ranking.py` (where the ranking actually happens).

## Mental model for using it well

- **Trust the verbatim default.** Don't try to "summarize before filing" — that's the move MemPalace is designed to make unnecessary.
- **Set up wings deliberately.** Wings are the coarsest filter; misclassified wings hurt recall more than over-chunked drawers.
- **Use the diary tools for agent-specific state.** Each agent's diary is its own namespace; this is how you avoid cross-agent memory contamination.
- **Add tunnels for cross-wing connections.** The graph is the navigation layer. If two wings are about related projects, a tunnel makes traversal cheap.
- **Run the hooks.** Background filing is what makes the "agent doesn't ask 'what are we doing today'" promise real. Without hooks you're filing manually.
- **Reach for rerank only when raw isn't enough.** Raw mode is free; rerank costs LLM calls. The benchmarks say raw is enough for most cases.
- **Back up the palace directory.** No built-in multi-device sync. A shared hub (`mempalace serve`) or a server backend lets several agents use one palace, but backups are still the user's job.

## When NOT to reach for this

- **Supersession matters.** If your domain needs "this fact replaced that fact," MemPalace's drawer search will resurrect the old one. The knowledge graph's `kg_supersede` (v3.6) handles single-valued facts you file there explicitly, but search over drawers has no notion of it. Reach for Neo (`Profile__Neo.md`) or a state-based engine (see Memgine in `Profile__StateBench.md`).
- **You need per-user-scope cross-session memory at multi-tenant scale.** MemPalace has wings, not tenants. Bolt scope on, or pick Mem0 which makes user scope mandatory.
- **You need cloud sync today.** Local-first by design. LanceDB never shipped; the closest options now are a self-hosted pgvector/Qdrant/Milvus backend or a hub on one machine that other agents reach over HTTP.
- **You want LLM-driven extraction.** That's the *not*-MemPalace pitch. The whole project argues you don't.
- **Markdown + git is enough.** As with the other entries: `context-vigilance` is a meaningfully simpler answer when the consumer is a human and the structure is the value.

## How this compares to the rest of the study

| Axis | MemPalace | Mem0 | Neo | Volt |
|---|---|---|---|---|
| **Write** | Verbatim, append-only, idempotent | LLM extraction (single-pass v3) | Append + deterministic supersession (cosine 0.85) | Verbatim immutable log + async LLM summary |
| **Recall** | Hybrid (vector + BM25 + closet-boost), zero-LLM default | Hybrid (vector + BM25 + entity-link), single-LLM extract on write | Semantic × confidence × success | DAG of summary nodes; raw still retrievable |
| **Supersession** | None on drawers (explicitly chosen); `kg_supersede` on the side graph since v3.6 | None in OSS | First-class | Implicit in time-ordered log |
| **Eviction** | None | None in OSS | Multiple coordinated mechanisms | Dolt: evict bindles; Upward: never |
| **Storage** | Chroma by default (five other backends opt-in) for drawers + closets, SQLite graph | Vector + entity store + SQLite | Scoped JSON files | Postgres DAG |
| **Scope axis** | Wing / Room | user_id / agent_id / run_id (required) | global / org / project / session (auto) | conversation / session |
| **Benchmark posture** | "Best-benchmarked" — published with methodology, including honesty about test-tuning | Strong on LoCoMo / LongMemEval / BEAM | Outcome detection on real codebases | OOLONG long-context claim |
| **Operational** | MCP (full + 3-tool light) + HTTP hub + CLI + library, local-first | Library / server / hosted | CLI + library + Claude/Codex plugins | Embedded coding agent |
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
| **Compression** | Closets (lossy pointer index) + opt-in AAAK, lossless drawers | Versioning + frontmatter |
| **Audit** | `filed_at` + `added_by` + immutable drawer | git log |

They compose cleanly: `context-vigilance` for the durable, reviewed project knowledge; MemPalace for the long tail of session transcripts, tool outputs, and verbatim conversation history that the agent should be able to fuzzy-recall without a human filing it. In a Lossless setup you could plausibly point a MemPalace wing at a `context-v/` directory and let agents query both surfaces — the markdown for structure, the palace for everything else.

The transferable lesson from MemPalace back to `context-vigilance`: **the closet-as-signal-never-gate pattern**. If we ever build a fuzzy index over `context-v/`, treat any summary layer as a ranking hint, not a filter. The underlying file should always be retrievable.

## Changes since the previous pin (2026-05-17 → 2026-11-22)

- **Pins:** `cabc3afb6eb2` → `b2ef8fed22f5` on `develop`, 1,094 non-merge commits (742 files changed). Releases in range: v3.3.6, v3.4.1, v3.5.0, v3.6.0, v3.7.0/3.7.1, v3.8.0, v3.9.0, v3.10.0, v3.11.0, v3.12.0 (2026-11-12, tagged on `main`), plus an unreleased block on `develop`. License unchanged (MIT).
- **The thesis holds and got stricter.** `CLAUDE.md:23-24` now leads with "Verbatim always" and adds that a crash mid-operation must leave the palace untouched. v3.12 stores user turns as written instead of spell-corrected, keeps non-UTF-8 sources byte-exact, and removed `mempalace.dedup`. On `develop` after v3.12, a conversation or diary re-mine journals the old drawer text to `drawer_revisions.jsonl` before overwriting it (`mempalace/revisions.py`), and `mempalace/retention.py` stops one miner pruning another's drawers. Each change closes a path where verbatim text could be lost.
- **Supersession arrived on the side graph, not on drawers.** v3.6.0 added `KnowledgeGraph.supersede()` / `mempalace_kg_supersede` (`mempalace/knowledge_graph.py:816`) to close one fact and open its successor at a shared instant. Drawer search still has no supersession, so the StateBench SFRR question in this profile stays open.
- **Backends became plural.** `sqlite_exact`, `rust_exact` (Rust engine in `crates/`), `milvus`, `qdrant`, and `pgvector` join Chroma (`README.md:189-210`, `mempalace/backends/`). Chroma remains the default.
- **New multi-agent coordination layer.** v3.7.0 added an append-only logstream (RFC 003) with tasks, handoffs, and artifacts, and v3.9 made one `mempalace serve` hub serve every agent on a machine. The MCP surface went from 29 to 47 tools, with a 3-tool light server on top (`mempalace/mcp_light_server.py`). This moves MemPalace from a memory store toward a shared brain for agent fleets, which the original profile did not cover.
- **Default embedder for new installs** changed from English-only `all-MiniLM-L6-v2` to multilingual `embeddinggemma-300m` ONNX (v3.3.6); existing palaces keep their model.
- **Code moved.** `searcher.py`, `palace.py`, and `mcp_server.py` became packages (`mempalace/searcher/`, `mempalace/palace/`, `mempalace/mcp_server/`); citations above are updated to the new paths.
- **Benchmarks:** headline numbers unchanged (96.6% raw R@5, 98.4% held-out, vendor-reported), but the README no longer headlines the 100% rerank figure (`README.md:255-260`).

## One-line summary

> MemPalace argues — and benchmarks — that the field's extraction-and-summarize memory pattern is over-engineered; that storing verbatim chunks in ChromaDB with hybrid semantic + BM25 ranking + a closet-as-signal layer outperforms the extraction systems on LongMemEval / LoCoMo / ConvoMem / MemBench while costing zero LLM calls at query time; and that "store everything, never lose information, rank cleverly" is a stronger baseline than anyone was measuring.
