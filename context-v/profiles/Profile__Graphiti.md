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
date_authored_current_draft: 2026-05-17
lede: >-
  `expired_at` is when the engine retired a fact; `invalid_at` is when it
  stopped being true in the world. Routinely different dates.
summary: >-
  Profile of Graphiti (Zep Software) as pinned in the memory-layers-for-agents
  study. It is the study's only genuine graph — typed nodes, typed edges,
  BFS-traversable, community-detectable — and the reference point whenever
  temporal correctness comes up. Covers the four interchangeable Cypher-flavoured
  backends (Neo4j / FalkorDB / Kuzu / Neptune), the five time fields on every
  edge, the episode-as-first-class-node design that keeps provenance one hop
  away, the six-stage add_episode pipeline, hybrid BM25 + cosine + BFS recall
  with RRF/MMR/cross-encoder rerankers, label-propagation community detection,
  and Pydantic entity types as the ontology surface. Note the benchmark posture:
  the numbers live in the arXiv Zep paper, not in a re-runnable in-repo harness.
publish: true
---

# Graphiti — Profile

A profile of Graphiti as it lives in this study (`studies/memory-layers-for-agents/graphiti/`). Cites pinned paths so you can jump to source rather than trust paraphrase. Read alongside [`Profile__Mem0.md`](./Profile__Mem0.md) and [`Profile__MemPalace.md`](./Profile__MemPalace.md) — Graphiti makes the opposite design bet from both, putting a real graph database (and the temporal model that goes with it) at the centre of the memory layer.

## TL;DR

Graphiti is the **open-source temporal knowledge-graph engine at the core of Zep's commercial context infrastructure** (`README.md:27-32`). It is the only entry in the study that is *actually* a graph: typed nodes, typed relationship edges, BFS-traversable, community-detectable, and **bi-temporally versioned** so every fact carries `valid_at`/`invalid_at`/`created_at`/`expired_at` and the system can answer "what did the agent believe at time T" — not just "what does the agent believe now."

It runs on four interchangeable Cypher-flavoured graph backends (`graphiti_core/driver/driver.py:59-63`): **Neo4j** (reference), **FalkorDB**, **Kuzu** (in-process), and **AWS Neptune**. Ingest happens through **Episodes** (text / message / JSON / fact-triple) → a single LLM call extracts entities + facts via Pydantic structured output → entity resolution + edge deduplication → episodic linking → optional community recomputation (label propagation) + summary. Recall is hybrid (`search/search.py:98-108`): BM25 fulltext + cosine on embeddings + BFS graph traversal, fused with RRF / MMR / cross-encoder rerankers. Custom **Pydantic entity types** are the ontology surface — far more flexible than Mem0's spaCy-NER-fixed labels.

The headline pitch is the temporal model. Where Mem0 ranks newer memories higher and hopes (`Profile__Mem0.md` §write policy), Graphiti stores both versions of a contradicting fact with explicit validity windows, so an agent can both retrieve the current truth and reason about what changed. The methodology is the subject of a peer-reviewed paper — *Zep: A Temporal Knowledge Graph Architecture for Agent Memory* (arXiv:2501.13956) — cited in `README.md:36-42`.

**One sentence:** *Graphiti is the only entry in the study that puts an actual typed, bi-temporal graph between the agent and its memories, with four pluggable graph backends, a single-LLM-call ingest pipeline, hybrid search with cross-encoder reranking, and a published architecture paper backing the temporal-knowledge-graph bet.*

## Why this exists — the design bet

Three observations drive the design:

1. **Vector recall flattens structure.** Vector + entity-link approaches (Mem0) and verbatim-chunk approaches (MemPalace) both collapse relationships into either implicit arrays or co-occurrence. Real questions ("who did Alice work with at her last company") need traversal, not just similarity.
2. **Facts have validity windows, not just timestamps.** "Alice works at Acme" was true; then it stopped being true. A single `updated_at` field can't represent that. Bi-temporal storage can.
3. **Every edge should trace back to its source episode.** No hallucinated relationships. Episodes are first-class nodes; edges remember which episodes produced them (`edges.py:263-285` — `episodes: list[str]`).

The methodology is documented in the *Zep* paper (arXiv:2501.13956) and validated against LongMemEval; the code in this submodule is the open-source engine that paper describes.

## Storage topology — four interchangeable graph backends

```python
class GraphProvider(Enum):
    NEO4J = 'neo4j'
    FALKORDB = 'falkordb'
    KUZU = 'kuzu'
    NEPTUNE = 'neptune'
```

(`graphiti_core/driver/driver.py:59-63`)

All four are Cypher-or-Cypher-like. Neo4j is the reference; **FalkorDB** is a Redis-module graph (fast, in-memory); **Kuzu** is an embedded analytical graph DB (no server); **Neptune** is AWS-managed. The driver abstraction wraps a thin `QueryExecutor` for pure queries plus a `GraphOperationsInterface` for provider-specific optimisations (e.g., fulltext index syntax differs per backend; the `fulltext_syntax` field on each driver papers over the difference).

Python 3.10+ required (`pyproject.toml:12`).

This is the most operationally-serious storage tier in the study. Mem0 leans on vector DBs; MemPalace on Chroma + SQLite; Neo on JSON files; Volt on embedded Postgres. Only Graphiti expects you to actually have a graph database — and gives you four options for which.

## The bi-temporal model — the headline design choice

Every edge carries five time fields (`edges.py:263-285`):

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

**The split between `expired_at` and `invalid_at` is the load-bearing detail.** `expired_at` is *system time* — when the engine decided to retire the record. `invalid_at` is *valid time* — when the underlying fact stopped being true in the world. They can be different. ("Alice left Acme on 2024-03-01" can be discovered and recorded in the graph on 2024-03-15.) Database-textbook bi-temporality, brought to agent memory.

The temporal model is indexed (`graph_queries.py`):

```
CREATE INDEX valid_at_episodic_index FOR (n:Episodic) ON (n.valid_at)
CREATE INDEX invalid_at_edge_index FOR ()-[e:RELATES_TO]-() ON (e.invalid_at)
```

So "show me facts that were valid at time T" is a real, indexed query — not a scan.

Compare to the rest of the study: Mem0 has `created_at`/`updated_at` only; MemPalace has `filed_at`; Neo has `created_at`/`last_accessed`/`access_count`; Volt has `created_at` on every immutable row. **Only Graphiti has both axes.**

## Node and episode schema

Three node types share a base (`nodes.py:93-99`):

```python
class Node(BaseModel, ABC):
    uuid: str
    name: str
    group_id: str           # multi-tenant partition
    labels: list[str]       # mutable type tags
    created_at: datetime
```

**`EpisodicNode`** (`nodes.py:318-332`) — the ingestion unit:

```python
source: EpisodeType                # message | json | text | fact_triple
source_description: str
content: str                       # raw episode payload
valid_at: datetime                 # document creation time — the temporal anchor
entity_edges: list[str]            # edge UUIDs the episode produced
episode_metadata: dict[str, Any]   # custom filters
```

**`EntityNode`** (`nodes.py:499-504`) — the extracted entity with an evolving regional summary:

```python
name_embedding: list[float] | None
summary: str                       # "regional summary of surrounding edges"
attributes: dict[str, Any]         # custom entity-type fields
```

**`CommunityNode`** (`nodes.py:687-689`) — clusters from label-propagation:

```python
name_embedding: list[float] | None
summary: str                       # cluster summary
```

The episode-as-first-class-node design is unusually disciplined. Mem0 and MemPalace bury their source material in a separate audit table. Graphiti keeps the episode *in the graph*, so traversal can reach from any edge to its provenance with a single hop.

## Write policy — the episode → graph pipeline

`add_episode(...)` (`graphiti.py:933-1016`) is a six-stage pipeline:

1. **Type coercion** — `EpisodeType.from_str()` validates source format.
2. **Combined extraction** — single LLM call via `extract_nodes_and_edges_bulk()` (`utils/bulk_utils.py`) with Pydantic structured output (`prompts/extract_nodes_and_edges.py`: `CombinedEntity`, `CombinedFact`).
3. **Deduplication** — `dedupe_nodes_bulk()` and `dedupe_edges_bulk()` compare against recent items using semantic similarity + LLM judgement (`prompts/dedupe_nodes.py`, `prompts/dedupe_edges.py`).
4. **Entity resolution** — `resolve_extracted_nodes()` / `resolve_extracted_edges()` merge with existing graph nodes.
5. **Episodic linking** — `build_episodic_edges()` creates `MENTIONS` edges from the episode to each extracted entity. Provenance preserved.
6. **Community recomputation** (optional) + **embedding generation** (async batch).

`add_episode_bulk()` (`graphiti.py:1183-1322`) is the throughput variant.

The whole pipeline runs **one LLM call per episode for extraction** plus dedup calls — comparable in cost to Mem0's v3 single-pass extraction but doing far more (typed entities + typed edges + provenance).

## Edge supersession — temporal, not pointer-based

This is the subtle bit and the right place to be skeptical. Graphiti does **not** automatically delete or invalidate superseded edges. Instead:

- At ingest, `dedupe_edges_bulk()` finds candidates that look like updates to existing edges; the system can set `valid_at` on the new edge and may backfill `invalid_at` on the old one (the code path is present but not aggressively documented).
- At query time, search filters can request `invalid_at IS NULL` to scope to currently-valid facts.
- No explicit `superseded_by`/`supersedes` pointer pair like Neo carries (`Profile__Neo.md` §write policy). Graphiti's bet is that **temporal windows + ranking** are enough.

This means StateBench's Superseded Fact Resurrection Rate (`Profile__StateBench.md` §metrics) is the metric to watch for Graphiti. The architectural prediction: the bi-temporal model *should* help (you can structurally filter for currently-valid facts), but the lack of explicit supersession pointers means correctness depends on dedup quality at ingest. Worth measuring rather than assuming.

## Recall API — hybrid with three modalities per element type

(`search/search.py:98-108`)

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

(`utils/maintenance/community_operations.py:93-138`)

Standard label-propagation:

1. Each node starts in its own community.
2. Each iteration: every node adopts the community of its edge-weighted plurality of neighbours.
3. Ties broken in favour of larger communities.
4. Repeat until convergence.

Operates on a projection (adjacency map with edge weights), so it's O(n × iterations), not O(n²).

Once communities form, an LLM is run over the member summaries to produce a **region-level abstract** (`community_operations.py:141-150`). Communities become a separate retrieval unit in the search config — useful for "summarise everything we know about Alice's network" queries.

## Custom entity types — Pydantic as the ontology

Custom entity types are arbitrary **Pydantic models** passed to the extraction prompt. The LLM assigns extracted entities to types by ID; the typed fields end up in the `attributes` dict on the resulting `EntityNode`. On recall, you filter by entity type.

This is the right level of ontology for an LLM-driven system: Pydantic enforces shape and validation; the LLM does the soft work of "is this entity a `Person` or a `Company`"; the graph stores both type and attributes. Compare to Mem0's spaCy-fixed PER/ORG/LOC (`Profile__Mem0.md` §schema) — Graphiti's ontology is open.

## Group / namespace model

Every node and edge carries a `group_id` (`nodes.py:93-99`). All searches scope to a list of `group_ids`. This is simpler than Mem0's three-level (`user_id`/`agent_id`/`run_id`) scoping and equivalent to MemPalace's wing concept. Multi-tenant isolation is at the partition-key level; no built-in RBAC beyond that.

## What's inside this submodule

| Path | What's there |
|---|---|
| `graphiti_core/graphiti.py` | Main `Graphiti` orchestration class |
| `graphiti_core/driver/` | Four-backend driver abstraction (Neo4j, FalkorDB, Kuzu, Neptune) |
| `graphiti_core/nodes.py` | Node base + Episodic / Entity / Community / Saga |
| `graphiti_core/edges.py` | Edge base + EntityEdge with bi-temporal fields |
| `graphiti_core/search/` | Hybrid search, configs, recipes, rerankers |
| `graphiti_core/prompts/` | LLM prompts (extract, dedupe nodes, dedupe edges, summarise) |
| `graphiti_core/llm_client/` | OpenAI / Anthropic / Gemini / Groq / Azure / generic / GLiNER2 |
| `graphiti_core/embedder/` | OpenAI / Azure / Gemini / Voyage / Sentence Transformers |
| `graphiti_core/cross_encoder/` | Cross-encoder rerankers (OpenAI / Gemini / BGE) |
| `graphiti_core/utils/maintenance/` | Community detection, edge ops, dedup helpers |
| `server/` | FastAPI REST wrapper (`/ingest`, `/search`, `/update`, `/delete`) |
| `mcp_server/` | MCP server exposing `search_memory` / `add_memory` / `update_memory` / `delete_memory` for Claude, Cursor, etc. |
| `examples/` | Quickstarts for Neo4j, FalkorDB, Neptune |
| `tests/` | Unit + integration tests (`_int` tests require Neo4j) |
| `spec/driver-operations-redesign.md` | Phase-2 refactor sketch (moving DB logic out of data models into namespace API) |
| `docker-compose.yml` | Bundled Neo4j stack |

If you read three files: `graphiti_core/graphiti.py` (the orchestrator), `graphiti_core/edges.py` (the temporal model in 30 lines), `graphiti_core/search/search.py` (where the hybrid pieces come together).

## LLM and embedder support

LLM clients (`graphiti_core/llm_client/`): OpenAI (default; structured output), Anthropic (Claude 3.5/3.7/4.5 with structured output via API), Gemini, Groq (edge extraction, no reasoning models), Azure OpenAI, generic OpenAI-compatible (local), GLiNER2 (lightweight NER alternative to full LLM).

Embedder clients (`graphiti_core/embedder/`): OpenAI (`text-embedding-3-small/large`), Azure, Gemini, Voyage, Sentence Transformers (local, no API key).

Default is OpenAI on both; swapping is straightforward.

## Operational story — three surfaces

1. **Library**: `pip install graphiti-core`. `from graphiti_core import Graphiti`. You bring a graph DB. Most flexible.
2. **REST server** (`server/`). FastAPI, Dockerised. Endpoints for ingest/search/update/delete. Multi-user via `group_id`.
3. **MCP server** (`mcp_server/`). Standard MCP tools (`search_memory`, `add_memory`, `update_memory`, `delete_memory`). Drop into Claude Desktop / Claude Code / Cursor and the agent gets graph memory.

All three share the same core; deployment is the only thing that changes.

## Benchmark posture

No benchmark code in the OSS repo. The headline numbers come from the *Zep* paper (arXiv:2501.13956, `README.md:36-42`) which evaluates the architecture on LongMemEval (long-horizon multi-turn agent memory) and reports it outperforming simpler vector-only approaches and older KG-based memory systems.

This means: claims are peer-reviewed and externally cited, but you can't trivially re-run them from this repo the way you can with Mem0's `evaluation/` or MemPalace's `benchmarks/`. Worth noting if benchmark reproducibility matters to your evaluation.

## Mental model for using it well

- **Pick the backend that matches your operational reality.** Kuzu for in-process (a single Python service). FalkorDB for low-latency Redis-flavoured deployments. Neo4j for everything-and-a-UI. Neptune if you're already on AWS.
- **Lean on the temporal model.** If two episodes contradict, ingest both. Let `valid_at` / `invalid_at` carry the story. Don't pre-deduplicate at the source — the graph is designed to hold the history.
- **Define custom entity types early.** Pydantic schemas you commit to up front shape the entire graph. Retrofitting types is painful.
- **Use `center_node_uuid` for personal queries.** "Tell me everything about Alice" should start at the Alice node and BFS out, not start with a query embedding.
- **Run community detection periodically, not per-ingest.** Label propagation isn't cheap; the docs and code path suggest it as a background job.
- **Filter `invalid_at IS NULL` for current-state queries.** Most production queries want "what's true now," not "what's ever been true."
- **Keep an eye on dedup quality.** Edge supersession in the OSS path depends on dedup correctness. Bad dedup → ghost facts.

## When NOT to reach for this

- **You don't want to run a graph DB.** Even Kuzu (embedded) is a more serious storage commitment than SQLite or a vector store. If the operational weight isn't worth it, MemPalace or Mem0 is simpler.
- **Your data has no real relationships.** A flat stream of journal entries doesn't need a graph. Use MemPalace and save the operational complexity.
- **You need open-source explicit supersession with provenance pointers.** Reach for Neo (`Profile__Neo.md`) — its `superseded_by`/`supersedes` model is more direct than Graphiti's temporal-windows-plus-dedup approach.
- **You want a code-reasoning agent.** Use Neo (memory for code) or Volt (whole coding agent). Graphiti is general; it doesn't specialise.
- **You want reproducible benchmark numbers in the repo.** The numbers exist in the paper; running them yourself takes more work than running Mem0's or MemPalace's harnesses.

## How this compares to the rest of the study

| Axis | Graphiti | Mem0 | MemPalace | Neo | Volt |
|---|---|---|---|---|---|
| **Storage** | Graph DB (Neo4j / FalkorDB / Kuzu / Neptune) | Vector + entity store + SQLite | Chroma + SQLite graph | Scoped JSON files | Postgres DAG |
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

## One-line summary

> Graphiti is the only entry in the study that puts a real, typed, bi-temporal graph between the agent and its memories — four interchangeable Cypher-flavoured backends, a single-LLM-call episode-to-graph extraction pipeline with Pydantic-typed entities, hybrid BM25 + cosine + BFS recall with cross-encoder reranking, label-propagation community detection, and a peer-reviewed paper validating the temporal-knowledge-graph architecture for agent memory.
