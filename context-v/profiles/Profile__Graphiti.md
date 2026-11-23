---
name: Graphiti Profile
slug: graphiti
upstream: https://github.com/getzep/graphiti
package: graphiti-core (PyPI)
license: Apache-2.0
maintainer: Zep Software, Inc. (Paul Paliychuk, Preston Rasmussen, Daniel Chalef)
study: studies/memory-layers-for-agents
profile_path: studies/memory-layers-for-agents/graphiti
profile_kind: library + mcp-server + rest-server
date_created: 2026-05-17
site_uuid: 20656636-87a3-4098-898f-754410e26a8f
hex_code: 8fu6wj
date_authored_initial_draft: 2026-05-17
date_authored_current_draft: 2026-11-23
lede: >-
  `expired_at` is when the engine retired a fact; `invalid_at` is when it
  stopped being true in the world. Routinely different dates.
summary: >-
  Profile of Graphiti (Zep Software) as pinned in the memory-layers-for-agents
  study. It is the study's only genuine graph — typed nodes, typed edges,
  BFS-traversable, community-detectable — and the reference point whenever
  temporal correctness comes up. Covers the four interchangeable Cypher-flavoured
  backends (Neo4j / FalkorDB / Neptune / Relate; Kuzu was removed in
  November 2026), the five time fields on every
  edge, the episode-as-first-class-node design that keeps provenance one hop
  away, the six-stage add_episode pipeline, hybrid BM25 + cosine + BFS recall
  with RRF/MMR/cross-encoder rerankers, label-propagation community detection,
  and Pydantic entity types as the ontology surface. Note the benchmark posture:
  the numbers live in the arXiv Zep paper, not in a re-runnable in-repo harness.
publish: true
title: Graphiti — Profile
date_modified: 2026-11-23
authors:
  - Michael Staton
status: Reference
type: Profiles
---

# Graphiti — Profile

A profile of Graphiti as it lives in this study (`studies/memory-layers-for-agents/graphiti/`). Cites pinned paths so you can jump to source rather than trust paraphrase. Read alongside [`Profile__Mem0.md`](./Profile__Mem0.md) and [`Profile__MemPalace.md`](./Profile__MemPalace.md) — Graphiti makes the opposite design bet from both, putting a real graph database (and the temporal model that goes with it) at the centre of the memory layer.

## TL;DR

Graphiti is the **open-source temporal knowledge-graph engine behind Zep's commercial context layer** (`README.md:79-92`; since August 2026 the README calls it "a framework for building temporal knowledge graphs" and adds a Zep-vs-Graphiti table at `README.md:98-115`). It is the only entry in the study that is *actually* a graph: typed nodes, typed relationship edges, BFS-traversable, community-detectable, and **bi-temporally versioned** so every fact carries `valid_at`/`invalid_at`/`created_at`/`expired_at` and the system can answer "what did the agent believe at time T" — not just "what does the agent believe now."

It runs on four interchangeable Cypher-flavoured graph backends (`graphiti_core/driver/driver.py:59-63`): **Neo4j** (reference), **FalkorDB**, **AWS Neptune**, and **Relate** (embedded, no server; added November 2026 to replace the removed Kuzu driver). Ingest happens through **Episodes** (text / message / JSON / fact-triple) → a single LLM call extracts entities + facts via Pydantic structured output → entity resolution + edge deduplication → episodic linking → optional community recomputation (label propagation) + summary. Recall is hybrid (`search/search.py:98`): BM25 fulltext + cosine on embeddings + BFS graph traversal, fused with RRF / MMR / cross-encoder rerankers. Custom **Pydantic entity types** are the ontology surface — far more flexible than Mem0's spaCy-NER-fixed labels.

The headline pitch is the temporal model. Where Mem0 ranks newer memories higher and hopes (`Profile__Mem0.md` §write policy), Graphiti stores both versions of a contradicting fact with explicit validity windows, so an agent can both retrieve the current truth and reason about what changed. The methodology is the subject of a peer-reviewed paper — *Zep: A Temporal Knowledge Graph Architecture for Agent Memory* (arXiv:2501.13956) — cited in `README.md:92`.

**One sentence:** *Graphiti is the only entry in the study that puts an actual typed, bi-temporal graph between the agent and its memories, with four pluggable graph backends, a single-LLM-call ingest pipeline (now with an opt-in concurrent mode), hybrid search with cross-encoder reranking, and a published architecture paper backing the temporal-knowledge-graph bet.*

## Why this exists — the design bet

Three observations drive the design:

1. **Vector recall flattens structure.** Vector + entity-link approaches (Mem0) and verbatim-chunk approaches (MemPalace) both collapse relationships into either implicit arrays or co-occurrence. Real questions ("who did Alice work with at her last company") need traversal, not just similarity.
2. **Facts have validity windows, not just timestamps.** "Alice works at Acme" was true; then it stopped being true. A single `updated_at` field can't represent that. Bi-temporal storage can.
3. **Every edge should trace back to its source episode.** No hallucinated relationships. Episodes are first-class nodes; edges remember which episodes produced them (`edges.py:228-260` — `episodes: list[str]`).

The methodology is documented in the *Zep* paper (arXiv:2501.13956) and validated against LongMemEval; the code in this submodule is the open-source engine that paper describes.

## Storage topology — four interchangeable graph backends

```python
class GraphProvider(Enum):
    NEO4J = 'neo4j'
    FALKORDB = 'falkordb'
    NEPTUNE = 'neptune'
    RELATE = 'relate'
```

(`graphiti_core/driver/driver.py:59-63`)

All four are Cypher-or-Cypher-like. Neo4j is the reference; **FalkorDB** is a Redis-module graph (fast, in-memory); **Neptune** is AWS-managed; **Relate** is an embedded graph store with native vector and temporal types (`graphiti_core/driver/relate_driver.py`, `graphiti_core/driver/relate/`, `pip install graphiti-core[relate]`, Python 3.11+). Kuzu, the embedded option at the May pin, was removed in #2151 (2026-11-14) after being marked deprecated. The driver abstraction wraps a thin `QueryExecutor` for pure queries plus a `GraphOperationsInterface` for provider-specific optimisations (e.g., fulltext index syntax differs per backend; the `fulltext_syntax` field on each driver papers over the difference).

Python 3.10+ required (`pyproject.toml:12`).

This is the most operationally-serious storage tier in the study. Mem0 leans on vector DBs; MemPalace on Chroma + SQLite; Neo on JSON files; Volt on embedded Postgres. Only Graphiti expects you to actually have a graph database — and gives you four options for which.

## The bi-temporal model — the headline design choice

Every edge carries five time fields (`created_at` from the base class, plus the four below; `edges.py:228-260`):

```python
class EntityEdge(Edge):
    name: str               # relation name (e.g. "WORKS_AT")
    fact: str               # self-contained natural-language description
    fact_embedding: list[float] | None
    episodes: list[str]     # episode UUIDs that produced this edge
    expired_at: datetime | None     # when Graphiti invalidated the record
    valid_at: datetime | None       # when the fact became true (real-world)
    invalid_at: datetime | None     # when the fact stopped being true
    reference_time: datetime | None # timestamp from the originating episode
    attributes: dict[str, Any]      # custom per-relation-type fields
```

**The split between `expired_at` and `invalid_at` is the load-bearing detail.** `expired_at` is *system time* — when the engine decided to retire the record. `invalid_at` is *valid time* — when the underlying fact stopped being true in the world. They can be different. ("Alice left Acme on 2024-03-01" can be discovered and recorded in the graph on 2024-03-15.) Database-textbook bi-temporality, brought to agent memory. Upstream made the distinction explicit in the field docs in November 2026 (c287b36): `expired_at` is now described as "when Graphiti marked the edge as invalidated (transaction time); use invalid_at to find when the fact stopped being true" (`edges.py:236-241`), and `invalid_at` notes that "soft deadlines are not invalid_at" (`edges.py:250-252`).

The temporal model is indexed (`graph_queries.py`):

```
CREATE INDEX valid_at_episodic_index FOR (n:Episodic) ON (n.valid_at)
CREATE INDEX invalid_at_edge_index FOR ()-[e:RELATES_TO]-() ON (e.invalid_at)
```

So "show me facts that were valid at time T" is a real, indexed query — not a scan.

Compare to the rest of the study: Mem0 has `created_at`/`updated_at` only; MemPalace has `filed_at`; Neo has `created_at`/`last_accessed`/`access_count`; Volt has `created_at` on every immutable row. **Only Graphiti has both axes.**

## Node and episode schema

Three node types (plus `SagaNode`, `nodes.py:894`, which groups an ordered series of episodes) share a base (`nodes.py:98-101`):

```python
class Node(BaseModel, ABC):
    uuid: str
    name: str
    group_id: str           # multi-tenant partition
    labels: list[str]       # mutable type tags
    created_at: datetime
```

**`EpisodicNode`** (`nodes.py:245`) — the ingestion unit:

```python
source: EpisodeType                # message | json | text | fact_triple
source_description: str
content: str                       # raw episode payload
valid_at: datetime                 # document creation time — the temporal anchor
entity_edges: list[str]            # edge UUIDs the episode produced
episode_metadata: dict[str, Any]   # custom filters
```

**`EntityNode`** (`nodes.py:426`) — the extracted entity with an evolving regional summary:

```python
name_embedding: list[float] | None
summary: str                       # "regional summary of surrounding edges"
attributes: dict[str, Any]         # custom entity-type fields
```

**`CommunityNode`** (`nodes.py:714`) — clusters from label-propagation:

```python
name_embedding: list[float] | None
summary: str                       # cluster summary
```

The episode-as-first-class-node design is unusually disciplined. Mem0 and MemPalace bury their source material in a separate audit table. Graphiti keeps the episode *in the graph*, so traversal can reach from any edge to its provenance with a single hop.

## Write policy — the episode → graph pipeline

`add_episode(...)` (`graphiti.py:1586`, with the stages in `_extract_and_resolve_nodes` at `:898` and `_extract_and_resolve_edges` at `:1020`) is a six-stage pipeline:

1. **Type coercion** — `EpisodeType.from_str()` validates source format.
2. **Combined extraction** — single LLM call via `extract_nodes_and_edges_bulk()` (`utils/bulk_utils.py`) with Pydantic structured output (`prompts/extract_nodes_and_edges.py`: `CombinedEntity`, `CombinedFact`).
3. **Deduplication** — `dedupe_nodes_bulk()` and `dedupe_edges_bulk()` compare against recent items using semantic similarity + LLM judgement (`prompts/dedupe_nodes.py`, `prompts/dedupe_edges.py`).
4. **Entity resolution** — `resolve_extracted_nodes()` / `resolve_extracted_edges()` merge with existing graph nodes.
5. **Episodic linking** — `build_episodic_edges()` creates `MENTIONS` edges from the episode to each extracted entity. Provenance preserved.
6. **Community recomputation** (optional) + **embedding generation** (async batch).

`add_episode_bulk()` (`graphiti.py:1890`) is the throughput variant; since v0.31.0 it defaults to combined extraction too.

Since #2154 (November 2026, after v0.31.2) there is also an opt-in **concurrent ingestion** mode for one graph: pass `concurrent_ingestion=ConcurrentIngestionConfig(...)` to `Graphiti(...)` (`graphiti.py:236`) and episodes run as units that produce change sets (`EndEdge`, `PatchNode`, `AddEpisodesToEdge`, …) committed in sequence through a state store with leases and stale-commit checks (`graphiti_core/ingestion/`). Before this, concurrent writes to one `group_id` could race on dedup.

Prompt and model selection now go through an instance-scoped `LLMRuntime` (`graphiti_core/llm_client/llm_runtime.py`, #1778), so a caller can override individual prompts or route stages to different models.

The whole pipeline runs **one LLM call per episode for extraction** plus dedup calls — comparable in cost to Mem0's v3 single-pass extraction but doing far more (typed entities + typed edges + provenance).

## Edge supersession — temporal, not pointer-based

This is the subtle bit and the right place to be skeptical. Graphiti does **not** delete superseded edges, and it only invalidates them when the LLM judges a contradiction. Instead:

- At ingest, edge resolution asks the LLM which existing edges the new fact contradicts; `resolve_edge_contradictions()` (`utils/maintenance/edge_operations.py:795-827`) then sets the old edge's `invalid_at` to the new edge's `valid_at` and stamps `expired_at` with the current time. The old edge stays in the graph.
- At query time, search filters can request `invalid_at IS NULL` to scope to currently-valid facts.
- No explicit `superseded_by`/`supersedes` pointer pair like Neo carries (`Profile__Neo.md` §write policy). Graphiti's bet is that **temporal windows + ranking** are enough.

This means StateBench's Superseded Fact Resurrection Rate (`Profile__StateBench.md` §metrics) is the metric to watch for Graphiti. The architectural prediction: the bi-temporal model *should* help (you can structurally filter for currently-valid facts), but the lack of explicit supersession pointers means correctness depends on dedup quality at ingest. Worth measuring rather than assuming.

## Recall API — hybrid with three modalities per element type

(`search/search.py:98`)

```python
async def search(
    clients: GraphitiClients,
    query: str,
    group_ids: list[str] | None,
    config: SearchConfig,
    search_filter: SearchFilters,
    center_node_uuid: str | None = None,
    bfs_origin_node_uuids: list[str] | None = None,
    query_vector: list[float] | None = None,
    driver: GraphDriver | None = None,
) -> SearchResults:
```

Per element type, three search methods:

| Element | Methods |
|---|---|
| Nodes | BM25 fulltext, cosine on name embedding, BFS |
| Edges | BM25 on fact text, cosine on fact embedding, BFS |
| Episodes | BM25 on raw content |
| Communities | BM25 on summary, cosine on embedding |

Rerankers (`search_config.py`):

- **RRF** — Reciprocal Rank Fusion of BM25 + semantic
- **MMR** — Maximal Marginal Relevance for diversity
- **Cross-encoder** — OpenAI / Gemini / BGE reranker via `graphiti_core/cross_encoder/`

Pre-built recipes in `search_config_recipes.py` give you sensible defaults (`COMBINED_HYBRID_SEARCH_RRF` etc.) without composing the config yourself.

The headline feature is `center_node_uuid` and `bfs_origin_node_uuids`: **search rooted in a specific node**, so "what do we know around Alice" becomes a graph traversal, not a fuzzy match. No other entry in the study can do this.

## Community detection — label propagation + LLM summarisation

(`utils/maintenance/community_operations.py:92-138`)

Standard label-propagation:

1. Each node starts in its own community.
2. Each iteration: every node adopts the community of its edge-weighted plurality of neighbours.
3. Ties broken in favour of larger communities.
4. Repeat until convergence.

Operates on a projection (adjacency map with edge weights), so it's O(n × iterations), not O(n²).

Once communities form, an LLM is run over the member summaries to produce a **region-level abstract** (`community_operations.py:140-195`). Communities become a separate retrieval unit in the search config — useful for "summarise everything we know about Alice's network" queries.

## Custom entity types — Pydantic as the ontology

Custom entity types are arbitrary **Pydantic models** passed to the extraction prompt. The LLM assigns extracted entities to types by ID; the typed fields end up in the `attributes` dict on the resulting `EntityNode`. On recall, you filter by entity type.

This is the right level of ontology for an LLM-driven system: Pydantic enforces shape and validation; the LLM does the soft work of "is this entity a `Person` or a `Company`"; the graph stores both type and attributes. Compare to Mem0's spaCy-fixed PER/ORG/LOC (`Profile__Mem0.md` §schema) — Graphiti's ontology is open.

## Group / namespace model

Every node and edge carries a `group_id` (`nodes.py:101`). All searches scope to a list of `group_ids`. This is simpler than Mem0's three-level (`user_id`/`agent_id`/`run_id`) scoping and equivalent to MemPalace's wing concept. Multi-tenant isolation is at the partition-key level; no built-in RBAC beyond that.

## What's inside this submodule

| Path | What's there |
|---|---|
| `graphiti_core/graphiti.py` | Main `Graphiti` orchestration class |
| `graphiti_core/driver/` | Four-backend driver abstraction (Neo4j, FalkorDB, Neptune, Relate) |
| `graphiti_core/ingestion/` | Concurrent ingestion for one graph: change sets, state stores, phases, scheduler (new since the May pin) |
| `graphiti_core/nodes.py` | Node base + Episodic / Entity / Community / Saga |
| `graphiti_core/edges.py` | Edge base + EntityEdge with bi-temporal fields |
| `graphiti_core/search/` | Hybrid search, configs, recipes, rerankers |
| `graphiti_core/prompts/` | LLM prompts (extract, dedupe nodes, dedupe edges, summarise) |
| `graphiti_core/llm_client/` | OpenAI / Anthropic / Gemini / Groq / Azure / generic / GLiNER2, plus `llm_runtime.py` and the Jev / OpenAI Decisions dedupe clients |
| `graphiti_core/embedder/` | OpenAI / Azure / Gemini / Voyage / Sentence Transformers |
| `graphiti_core/cross_encoder/` | Cross-encoder rerankers (OpenAI / Gemini / BGE) |
| `graphiti_core/utils/maintenance/` | Community detection, edge ops, dedup helpers |
| `server/` | Reference FastAPI REST wrapper (`/ingest`, `/search`, `/update`, `/delete`) |
| `mcp_server/` | Reference MCP server (v1.1.3, MCP SDK 2.x) exposing `add_memory`, `search_nodes`, `search_memory_facts`, `get_episodes`, `get_entity_edge`, `delete_episode`, `delete_entity_edge`, `summarize_saga`, `build_communities` (`mcp_server/src/graphiti_mcp_server.py:402-930`) |
| `examples/` | Quickstarts for Neo4j, FalkorDB, Neptune |
| `tests/` | Unit + integration tests (`_int` tests require Neo4j) |
| `spec/driver-operations-redesign.md` | Phase-2 refactor sketch (moving DB logic out of data models into namespace API) |
| `docker-compose.yml` | Bundled Neo4j stack |

If you read three files: `graphiti_core/graphiti.py` (the orchestrator), `graphiti_core/edges.py` (the temporal model in 30 lines), `graphiti_core/search/search.py` (where the hybrid pieces come together).

## LLM and embedder support

LLM clients (`graphiti_core/llm_client/`): OpenAI (default; structured output; v0.31.0 replaced the deprecated `gpt-4.1-nano` defaults), Anthropic (documented model IDs moved to Claude 5 in this range; the client no longer sends `temperature` and logs a warning when one is set), Gemini, Groq (edge extraction, no reasoning models), Azure OpenAI, generic OpenAI-compatible (local), GLiNER2 (lightweight NER alternative to full LLM).

Embedder clients (`graphiti_core/embedder/`): OpenAI (`text-embedding-3-small/large`), Azure, Gemini, Voyage, Sentence Transformers (local, no API key).

Default is OpenAI on both; swapping is straightforward.

## Operational story — three surfaces

1. **Library**: `pip install graphiti-core`. `from graphiti_core import Graphiti`. You bring a graph DB. Most flexible.
2. **REST server** (`server/`). FastAPI, Dockerised. Endpoints for ingest/search/update/delete. Multi-user via `group_id`.
3. **MCP server** (`mcp_server/`). MCP tools for adding episodes, searching nodes and facts, reading and deleting episodes and edges, saga summaries, and community builds. Drop into Claude Desktop / Claude Code / Cursor and the agent gets graph memory. Since #1926 each tool call is routed to the graph of the `group_id` it names.

All three share the same core; deployment is the only thing that changes. Since October 2026 the README labels the two servers **reference implementations, not production deployments**: no authentication, no authorization, and any caller can read or change any `group_id`. Both now listen on localhost by default (#2032). For production, upstream points to Zep.

## Benchmark posture

No benchmark code in the OSS repo. The headline numbers come from the *Zep* paper (arXiv:2501.13956, `README.md:92`) which evaluates the architecture on LongMemEval (long-horizon multi-turn agent memory) and reports it outperforming simpler vector-only approaches and older KG-based memory systems.

This means: claims are peer-reviewed and externally cited, but you can't trivially re-run them from this repo the way you can with Mem0's `evaluation/` or MemPalace's `benchmarks/`. Worth noting if benchmark reproducibility matters to your evaluation.

## Mental model for using it well

- **Pick the backend that matches your operational reality.** Relate for in-process (a single Python service; Kuzu filled this slot until November 2026). FalkorDB for low-latency Redis-flavoured deployments. Neo4j for everything-and-a-UI. Neptune if you're already on AWS.
- **Lean on the temporal model.** If two episodes contradict, ingest both. Let `valid_at` / `invalid_at` carry the story. Don't pre-deduplicate at the source — the graph is designed to hold the history.
- **Define custom entity types early.** Pydantic schemas you commit to up front shape the entire graph. Retrofitting types is painful.
- **Use `center_node_uuid` for personal queries.** "Tell me everything about Alice" should start at the Alice node and BFS out, not start with a query embedding.
- **Run community detection periodically, not per-ingest.** Label propagation isn't cheap; the docs and code path suggest it as a background job.
- **Filter `invalid_at IS NULL` for current-state queries.** Most production queries want "what's true now," not "what's ever been true."
- **Keep an eye on dedup quality.** Edge supersession in the OSS path depends on dedup correctness. Bad dedup → ghost facts.
- **Turn on concurrent ingestion before running parallel writers against one graph.** Without `ConcurrentIngestionConfig`, parallel `add_episode` calls on one `group_id` resolve against each other's half-written state.

## When NOT to reach for this

- **You don't want to run a graph DB.** Even Relate (embedded) is a more serious storage commitment than SQLite or a vector store. If the operational weight isn't worth it, MemPalace or Mem0 is simpler.
- **Your data has no real relationships.** A flat stream of journal entries doesn't need a graph. Use MemPalace and save the operational complexity.
- **You need open-source explicit supersession with provenance pointers.** Reach for Neo (`Profile__Neo.md`) — its `superseded_by`/`supersedes` model is more direct than Graphiti's temporal-windows-plus-dedup approach.
- **You want a code-reasoning agent.** Use Neo (memory for code) or Volt (whole coding agent). Graphiti is general; it doesn't specialise.
- **You want reproducible benchmark numbers in the repo.** The numbers exist in the paper; running them yourself takes more work than running Mem0's or MemPalace's harnesses.

## How this compares to the rest of the study

| Axis | Graphiti | Mem0 | MemPalace | Neo | Volt |
|---|---|---|---|---|---|
| **Storage** | Graph DB (Neo4j / FalkorDB / Neptune / Relate) | Vector + entity store + SQLite | Chroma + SQLite graph | Scoped JSON files | Postgres DAG |
| **Relationships** | Explicit typed bi-temporal edges | Implicit (entity-link arrays) | None (flat) | `depends_on` between facts | Implicit (DAG over messages) |
| **Supersession** | Temporal windows + dedup-at-ingest | None in OSS | None (and explicitly chosen) | Explicit `superseded_by` / cascade | Implicit (immutable log) |
| **Ontology** | Custom Pydantic entity types | spaCy NER fixed | None | Typed fact kinds | None |
| **Recall** | BM25 + cosine + BFS + RRF/MMR/cross-encoder | Semantic + BM25 + entity boost | Semantic + BM25 + closet boost | Semantic × confidence × success | DAG of summary nodes + `lcm_grep` |
| **LLM calls per write** | One (combined extraction) | One (v3 single-pass) | Zero (verbatim) | Zero (deterministic supersession) | One per summary at thresholds |
| **Benchmark posture** | Peer-reviewed paper (arXiv:2501.13956); no in-repo harness | LoCoMo / LongMemEval / BEAM in `evaluation/` | LongMemEval / LoCoMo / ConvoMem / MemBench in `benchmarks/` | Outcome detection on real codebases | OOLONG long-context claim |
| **Best fit** | Domains with real relationships + temporal change | General agent memory across users | Long-tail verbatim recall | Code-reasoning loops | Long-horizon coding |

The interesting comparison is **Graphiti vs MemPalace**: opposite bets on structure. MemPalace argues structure is over-engineered and pure verbatim + good ranking wins. Graphiti argues structure is *under*-engineered and you need a real graph + temporal model. Both have benchmarks behind them; the methodologies don't overlap cleanly enough to pick a winner from numbers alone. The real answer depends on whether your domain has relationships that matter.

## How this compares to our own `context-vigilance` skill

`context-vigilance` uses wikilinks (`[[other-doc]]`) as its relationship surface and has no concept of temporal edges. A wikilink is a hyperlink — it doesn't carry "this link became true at T1 and stopped being true at T2." Graphiti is structurally what you'd build if you wanted `context-vigilance`'s wikilink graph to be *queryable*, *temporally aware*, and *traversable by an agent* without grep.

You wouldn't replace `context-vigilance` with Graphiti — the human-readable markdown layer is irreplaceable for human review. But if we ever needed to make wikilink relationships first-class to an agent (e.g., "find every spec that depends on a blueprint that was superseded after 2025-Q1"), Graphiti's data model is the shape that question wants. The transferable lesson is **bi-temporal edge metadata**: `valid_at` and `invalid_at` as separate fields. If we ever extend frontmatter to model deprecation, that's the schema to copy.

## Changes since the previous pin (2026-05-14 → 2026-11-22)

- **Pins:** `9a2d6d02bf0d` → `45fdf546c74d` on `main`, 256 non-merge commits (115 are CLA signatures). Releases in range: graphiti-core v0.29.1–v0.29.3, v0.30.0–v0.30.2, v0.31.0–v0.31.2 (`pyproject.toml` reads 0.31.2); MCP server mcp-v1.1.0–mcp-v1.1.3. The newest commits (Relate driver, concurrent ingestion, Kuzu removal) are on `main` after v0.31.2 and not yet in a tagged core release. License unchanged (Apache-2.0).
- **The temporal model is unchanged and now documented the way this profile reads it.** c287b36 rewrote the `expired_at` description to "transaction time; use invalid_at to find when the fact stopped being true" (`graphiti_core/edges.py:236-241`). The lede of this profile now matches upstream's own wording. The Zep-fork port (#1964) also removed a duplicate `reference_time` field on `EntityEdge`.
- **Backends: Kuzu out, Relate in.** #2151 removed the Kuzu driver; #2158 added Relate, an embedded store (`graphiti_core/driver/relate_driver.py`, `GraphProvider.RELATE` at `driver/driver.py:63`). The four-backend count holds, with a different embedded option.
- **Concurrent ingestion for one graph** (#2154, `graphiti_core/ingestion/`). Episodes become units that emit change sets committed in order through a leased state store. This addresses parallel writers on one `group_id`, which the May profile did not discuss.
- **Extraction and dedup moved closer to Zep's hosted engine.** #1964 upstreamed "extraction, dedup, and temporal fixes from the Zep fork"; #1997 made a new combined-extraction prompt the default; #1984 added Jev and OpenAI Decisions clients for node and edge dedupe; #1778 added `LLMRuntime` for per-instance prompt and model routing. `remove_episode` now keeps facts that other episodes restated and rebuilds node summaries (#2025, `graphiti.py:2523`).
- **Servers are explicitly reference-only.** The README (f71d140, #2029) says the MCP and REST servers have no auth and are not production deployments; #2032 binds them to localhost by default. The MCP server moved to MCP SDK 2.x (#1921).
- **Positioning:** the README was reframed (#1801, #2053) as "a framework for building temporal knowledge graphs," with a Zep-vs-Graphiti table that names Zep's own graph service, Konig (`README.md:98-115`).

## One-line summary

> Graphiti is the only entry in the study that puts a real, typed, bi-temporal graph between the agent and its memories — four interchangeable Cypher-flavoured backends, a single-LLM-call episode-to-graph extraction pipeline with Pydantic-typed entities, hybrid BM25 + cosine + BFS recall with cross-encoder reranking, label-propagation community detection, and a peer-reviewed paper validating the temporal-knowledge-graph architecture for agent memory.
