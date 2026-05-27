---
name: Honcho Profile
slug: honcho
upstream: https://github.com/plastic-labs/honcho
package: honcho (server), honcho-ai (PyPI SDK), @honcho-ai/sdk (npm SDK)
license: AGPL-3.0
maintainer: Plastic Labs
study: studies/memory-layers-for-agents
profile_path: studies/memory-layers-for-agents/honcho
profile_kind: library + server + hosted-platform + MCP worker
date_created: 2026-05-26
---

# Honcho — Profile

A profile of Honcho as it lives in this study (`studies/memory-layers-for-agents/honcho/`). Read this alongside [[Profile__Mem0]] and [[Profile__Letta]] — the three define almost the whole "general-purpose agent memory" frontier. Honcho is the one that takes **theory-of-mind** seriously enough to put it in the schema.

## TL;DR

Honcho is **multi-peer memory infrastructure** built around an `(observer, observed)` keying of all derived knowledge. Storage is Postgres with `pgvector` (`pyproject.toml:15`, `database/init.sql:1`); the load-bearing tables are `messages` (immutable append-only, FTS-indexed) (`src/models.py:206-273`) and `collections` (`src/models.py:334-375`) — and every collection is **unique on `(observer, observed, workspace_name)`** (`src/models.py:357-362`), which is the single line of code that distinguishes Honcho from every other system in this study.

The write path is **append-only messages + async background reasoning**. Messages enqueue queue items (`src/deriver/enqueue.py:53-72`); a separate **deriver worker process** (`uv run python -m src.deriver`, `src/deriver/__main__.py:73-85`) consumes them and runs a **single structured-output LLM call per batch** (`src/deriver/deriver.py:36-60`) that emits explicit conclusions into `(observer, observed)` collections. A scheduled **Dreamer** (`src/dreamer/orchestrator.py:67-100`) wakes up off-peak and runs a two-phase **DeductionSpecialist → InductionSpecialist** consolidation pass that produces deductive and inductive conclusions, with surprisal-based prioritization (`src/dreamer/surprisal.py:46-60`). The API never blocks on any of this.

Recall is a **tool-using Dialectic agent** (`src/dialectic/core.py:52-100`) — the only synchronous LLM call on the request path. It loops over a tool set (`search_memory`, `search_messages`, `get_observation_context`, `grep_messages`, `get_messages_by_date_range`, `search_messages_temporal`, `get_reasoning_chain`) at one of five reasoning tiers (`minimal/low/medium/high/max`) until it has enough context to answer, then synthesizes natural-language output. Search itself is **hybrid** — Postgres full-text-search (GIN index on `to_tsvector('english', content)`, `src/models.py:264-268`) fused with HNSW cosine similarity on pgvector via Reciprocal Rank Fusion (`src/utils/search.py:36-75`).

If you want one sentence: **Honcho is a multi-peer Postgres+pgvector memory system where storage is keyed by `(observer, observed)`, ingest is async (a deriver worker plus a scheduled dreamer), and recall is a tool-using Dialectic agent over hybrid FTS+vector search — the only system in this study that natively models "what does peer A understand about peer B?"**

## Why this exists — the design bet

Most agent-memory systems index by user. Honcho indexes by *relationship*. The thesis (`CLAUDE.md:14-25`, "Peer Paradigm") is that "users" and "agents" are the same kind of object — **peers** — and the interesting unit of memory is not "what does the system know about Alice" but "what does Alice's representation-within-the-coach-agent look like, and how does that differ from the same coach's representation within Alice's session with the therapist?"

1. **`(observer, observed)` is the indexing axis, not a metadata field.** The `Collection` table is unique on `(observer, observed, workspace_name)` (`src/models.py:357-362`). Every derived conclusion (`Document`) lives in exactly one collection (`src/models.py:428-435`). Self-representation is just the special case `observer == observed`.
2. **The write path is asynchronous on purpose.** Messages return from the API immediately (`src/deriver/enqueue.py:53-72`); reasoning happens in a separate worker process so the synchronous path never blocks on LLM latency. The README's own caveat — "newly-added messages may take a moment to be reflected in chat/representation responses" (`README.md:154`) — is the explicit cost.
3. **The deriver is deliberately "minimal."** A single structured-output LLM call per batch, not an agentic tool loop (`src/deriver/deriver.py:36-60`, `CLAUDE.md:101-110`). The agentic loop is reserved for two places where it earns its keep: the Dialectic on read (`src/dialectic/core.py:52-100`) and the Dreamer specialists off-cycle (`src/dreamer/specialists.py:74-80`).
4. **Conclusions form a reasoning tree.** Each `Document` carries `source_ids` (premise documents) and a `level` ∈ `{explicit, deductive, inductive, contradiction}` (`src/models.py:386-395`, `src/utils/types.py:240`). The Dialectic can traverse the chain via `get_reasoning_chain` (`src/utils/agent_tools.py:790`).
5. **Sessions are many-to-many with peers via a temporal membership table.** `session_peers` carries `joined_at` / `left_at` (`src/models.py:73-82`), and recall filters messages by whether a peer was *actually present* at the time the message was created (`src/utils/search.py:190-245`). This is what makes "what did Alice know at the time" answerable, not just "what's in the corpus now."

## Storage topology

| Layer | Backend | Purpose | Code |
|---|---|---|---|
| Primary OLTP | Postgres 15+ with `pgvector` extension | All resources (workspaces, peers, sessions, messages, collections, documents, queue) | `pyproject.toml:15`, `database/init.sql:1`, `docker-compose.yml.example:74-91` |
| Vector | `pgvector` HNSW indexes on `MessageEmbedding.embedding` and `Document.embedding` (m=16, ef_construction=64, cosine_ops) | Semantic search over messages and conclusions | `src/models.py:317-324`, `src/models.py:451-460` |
| Full-text | Postgres GIN on `to_tsvector('english', content)` | Keyword/BM25-like search over message content | `src/models.py:264-268` |
| Cache / async coordination | Redis (`cashews` client) | Hot-path caching, queue scheduling, session locks | `pyproject.toml:37-38`, `src/cache/client.py` |
| Pluggable external vector | Turbopuffer or LanceDB (optional) | Swap pgvector out for managed/embedded vector stores | `pyproject.toml:34-35`, `src/vector_store/__init__.py:53,213`, `src/vector_store/{turbopuffer,lancedb}.py` |
| Queue | `queue` table in same Postgres | Background tasks (representation, summary, dream, webhook, deletion, reconciler) | `src/models.py:477-529` |
| MCP edge | Cloudflare Worker | Remote MCP server at `mcp.honcho.dev` | `mcp/README.md`, `mcp/wrangler.toml` |

Note what's **not** in the stack: no separate graph DB, no Neo4j, no Elasticsearch, no separate object store. The whole system is "Postgres + Redis + a worker process." That is the operational selling point.

## Schema of a memory record

The two load-bearing tables, with the columns that matter for memory semantics.

**`collections`** — the `(observer, observed)` keying (`src/models.py:334-375`):

```python
class Collection(Base):
    id: str                  # nanoid
    observer: str            # peer name, indexed
    observed: str            # peer name, indexed
    workspace_name: str
    h_metadata: JSONB
    internal_metadata: JSONB
    # UNIQUE(observer, observed, workspace_name)
```

**`documents`** — the conclusion itself (`src/models.py:378-473`):

```python
class Document(Base):
    id: str                       # nanoid
    content: str                  # the conclusion text
    level: DocumentLevel          # "explicit" | "deductive" | "inductive" | "contradiction"
    times_derived: int            # how many dream cycles have re-derived this
    embedding: Vector(_VECTOR_DIM)
    source_ids: list[str] | None  # premise document IDs (reasoning tree)
    observer: str
    observed: str
    workspace_name: str
    session_name: str | None
    deleted_at: datetime | None   # soft delete
    sync_state: "synced|pending|failed"   # embedding pipeline state
    created_at: datetime
```

**`messages`** — raw, immutable, FTS-indexed (`src/models.py:206-273`):

```python
class Message(Base):
    id: BigInt                # autoincrement primary key
    public_id: str            # nanoid, exposed via API
    session_name: str
    peer_name: str            # who said it
    workspace_name: str
    content: str              # max 65535 chars
    seq_in_session: BigInt    # ordering within session
    token_count: int
    created_at: datetime
    # GIN(to_tsvector('english', content))  -- the FTS index
    # composite FK -> sessions, peers
```

The four `DocumentLevel` values are the part that doesn't exist in Mem0 or Letta:

- **`explicit`** — extracted directly from a message by the deriver (`src/deriver/prompts.py:39-80`).
- **`deductive`** — logical necessity inferred by the DeductionSpecialist during a dream (`src/dreamer/specialists.py`, `src/utils/agent_tools.py:824-833`).
- **`inductive`** — pattern across many explicit/deductive facts, produced by the InductionSpecialist (`src/utils/agent_tools.py:839-846`).
- **`contradiction`** — conflict flagged for resolution (`src/utils/types.py:240`).

The reasoning chain is reified in the data — `source_ids` is GIN-indexed for tree traversal (`src/models.py:462-466`). [[Profile__Mem0]]'s entity-link array is the closest analog, but it doesn't carry level or premise semantics.

## Write policy — append-only messages + async deriver + scheduled dreamer

The write path has **three temporally separated stages**:

**1. API enqueue (synchronous, fast).** `POST /v3/.../messages` (or `add_messages` from the SDK) writes immutable rows into `messages` and inserts queue items into `queue` (`src/deriver/enqueue.py:53-72`). Returns immediately.

**2. Minimal deriver (async, per-batch).** A separate worker process (`uv run python -m src.deriver`, `src/deriver/__main__.py`) consumes `queue` items grouped by **work unit key** — `representation:{workspace}:{session}:{observed}` (`src/utils/work_unit.py:53-57`). For each batch it makes **a single LLM call** with the `minimal_deriver_prompt` (`src/deriver/prompts.py:39-80`) and structured output, extracting explicit atomic facts about the `observed` peer. Each extracted observation becomes a `Document` row in every collection where some other peer observes this one (`src/deriver/deriver.py:36-60`). This is a deliberate trade — `CLAUDE.md:101-110` calls this "minimal deriver" and contrasts it with an agentic loop: "trades flexibility for cost and predictability."

**3. Dreamer (scheduled, off-peak).** A `DreamScheduler` (`src/dreamer/dream_scheduler.py:33-80`) accumulates pending dreams per `(observer, observed)` and fires them on a delay after activity stops (cancelled if the peer becomes active again, `src/deriver/enqueue.py:33-51`). A dream run (`src/dreamer/orchestrator.py:67-100`) is two specialist phases:
- **DeductionSpecialist** (`src/dreamer/specialists.py`, tools at `src/utils/agent_tools.py:824-833`): an agentic loop that creates deductive conclusions from explicit ones, can delete duplicates, and writes to the **peer card** (a small identity-marker list capped at 40 entries × 200 chars, `src/utils/agent_tools.py:34-46`).
- **InductionSpecialist** (`src/utils/agent_tools.py:839-846`): creates inductive patterns from explicit + deductive conclusions. Explicitly forbidden from writing the peer card (`CHANGELOG.md:38`).

Surprisal scoring (`src/dreamer/surprisal.py:46-60`) pre-filters which observations the specialists focus on — high-surprisal facts (geometrically distant from the existing tree) get prioritized.

The Summarizer (`src/utils/summarizer.py:1-80`, `CLAUDE.md:133-138`) runs alongside as a direct LLM call (no tools), producing two-tier session summaries: **short** every 20 messages, **long** every 60 messages.

**There is no supersession primitive in the schema.** A contradicted conclusion gets a sibling `contradiction`-level document rather than a `superseded_by` pointer (compare `[[Profile__Neo]]` §schema). The Dreamer can `delete_observations` (soft delete via `deleted_at`, `src/models.py:406-408`) but the API user can also `DELETE /v3/.../conclusions/{id}` directly (`src/routers/conclusions.py:134-139`). State-correctness over time is therefore the Dreamer's job, not the schema's.

## Recall API

Two distinct shapes, with very different cost profiles:

**Low-latency hybrid search** (`peer.search(...)`, `session.search(...)`, `honcho.search(...)`) — no LLM call. Pgvector HNSW + Postgres FTS, fused with Reciprocal Rank Fusion (`src/utils/search.py:36-75`):

```python
RRF_score = sum(1 / (k + rank_i))   # k=60 default
```

The vector search oversamples by 2× to handle deduplication across chunked embeddings without breaking the HNSW index scan (`src/utils/search.py:175-187`). FTS falls back to ILIKE when the query has special characters (`src/utils/search.py:266-308`).

**Agentic Dialectic** (`peer.chat(query=...)`, `POST /v3/.../peers/{peer_id}/chat`) — the headline endpoint. A tool-using LLM agent (`src/dialectic/core.py:52-100`) runs a loop over the following tools (`src/utils/agent_tools.py:782-791`):

| Tool | What it does |
|---|---|
| `search_memory` | Vector search over `documents` in `(observer, observed)` collection |
| `search_messages` | Hybrid search over `messages` |
| `get_observation_context` | Fetch the original messages a conclusion came from |
| `grep_messages` | Exact-text ILIKE for names/dates/keywords |
| `get_messages_by_date_range` | Temporal slicing |
| `search_messages_temporal` | Semantic + date filtering combined |
| `get_reasoning_chain` | Traverse `source_ids` to walk the deductive tree |

The agent has **five reasoning tiers** (`minimal/low/medium/high/max`) each with its own model config and tool set — `minimal` reduces to just `search_memory` + `search_messages` (`src/utils/agent_tools.py:795-798`) for cost. The default returns natural language; streaming is supported (`src/dialectic/chat.py`).

**Peer-perspective filtering** (`src/utils/search.py:190-245`) is the part most other systems can't do: messages from a session are only returned if the peer was a member at the time the message was created. Time-windowed visibility, structurally enforced.

## Eviction & compaction

There is **no automatic TTL**. The cleanup primitives that exist:

- **`Document.deleted_at`** (`src/models.py:406-408`) — soft delete. The Dreamer's `delete_observations` tool uses this (`src/utils/agent_tools.py:809`).
- **Two-tier summarization** (`src/utils/summarizer.py`, `CLAUDE.md:133-138`) — short summary every 20 messages, long every 60. Reduces the working context size for the Dialectic without deleting source messages.
- **Reconciler** (`src/reconciler/`, hosted by the deriver worker, `CLAUDE.md:79-87`) — embeds `MessageEmbedding` rows with `sync_state='pending'` and removes stale queue items. This is a *consistency* loop, not a *forgetting* loop.
- **Dream-driven consolidation** (`src/dreamer/specialists.py`) — DeductionSpecialist can mark duplicates for deletion. This is the closest Honcho gets to active forgetting.
- **`times_derived`** counter on `Document` (`src/models.py:389-391`) — tracks how often a conclusion has been re-derived. Available to the Dreamer for relevance scoring but no automatic demotion policy.

Compare [[Profile__Volt]]'s deterministic three-level escalation or [[Profile__Neo]]'s `prune_stale_facts`. Honcho's bet is that consolidation (deduction collapses redundancy, induction abstracts) is enough — and that you should preserve message history rather than compress it.

## Scopes & namespacing

A four-level hierarchy enforced by composite foreign keys (`CLAUDE.md:172-176` "Composite-FK multi-tenancy"):

```
Workspace                              # tenant root
  └── Peer (uniq on name, workspace)   # human OR agent — same type
        └── Session (m2m via session_peers, temporal joined_at/left_at)
              └── Message (uniq on workspace, session, seq_in_session)
                    └── (after deriver) Document in Collection(observer, observed)
```

`workspace_name` participates in nearly every composite FK (`src/models.py:243-251, 309-316, 364-374, 427-450`), which makes cross-workspace data leakage structurally impossible at the schema level. This is stronger than [[Profile__Mem0]]'s metadata-filter scope and roughly equivalent to [[Profile__Letta]]'s actor-based namespacing.

The **peer observation switches** are per-session, per-peer (`src/schemas/configuration.py:207-220`):

- `observe_me` — whether Honcho should form *any* representation of this peer.
- `observe_others` — whether this peer forms session-level representations of other peers (i.e., creates `(this_peer, other_peer)` collections).

That `observe_others` flag is the explicit theory-of-mind switch. Off by default; flip it on per-session when you want a coach-agent that models how a user thinks about their therapist.

## Operational story

Two cooperating processes that share Postgres + Redis (`CLAUDE.md:96-100`):

1. **API server** — `uv run fastapi dev src/main.py` (`pyproject.toml:11`, `src/main.py:1-40`). Handles HTTP, enqueues background work, hosts the **Dialectic** agent inline on `/chat`.
2. **Deriver worker** — `uv run python -m src.deriver` (`src/deriver/__main__.py:73-85`), uvloop-based. Long-running queue consumer running the Deriver, Summarizer, and Dreamer off `queue` rows. Scalable horizontally via `DERIVER_WORKERS`. Hosts an in-process `ReconcilerScheduler` (`src/reconciler/`).

The reference Docker stack (`docker-compose.yml.example`) is exactly four services: `api` (port 8000), `deriver`, `database` (pgvector/pgvector:pg15), `redis` (redis:8.2). Optional Prometheus + Grafana for observability. The whole thing runs locally with `docker compose up -d --build`.

Three deployment modes, one API:

1. **Managed** — `api.honcho.dev`. SDK defaults here. `MemoryClient`-style usage via `Honcho(workspace_id=..., api_key=...)` (`README.md:83-115`).
2. **Self-hosted** — clone the repo, copy `docker-compose.yml.example`, `docker compose up` (`README.md:73`).
3. **MCP edge** — a Cloudflare Worker (`mcp/README.md:1-50`) exposes the whole API surface as MCP tools at `mcp.honcho.dev`. Self-hosters point it at their own deployment via `HONCHO_API_URL` (`CHANGELOG.md:23`).

Telemetry is CloudEvents-based (`src/telemetry/events/`, `CLAUDE.md:131-135`), with `LLMCallCompletedEvent` carrying full cost attribution per provider hit (`CHANGELOG.md:16`). Langfuse and Sentry are first-class (`pyproject.toml:16,26`).

**License is AGPL-3.0** (`LICENSE:1-2`). Network-server copyleft: any modified version exposed over a network must publish source. The managed `api.honcho.dev` exists in part to give commercial users a way to consume Honcho without taking on AGPL obligations — this is the explicit business model. Compare [[Profile__Mem0]] (Apache-2.0, no copyleft) and [[Profile__Letta]].

## What's inside this submodule

| Path | What's there |
|---|---|
| `src/models.py` | SQLAlchemy ORM — the load-bearing schema (579 lines) |
| `src/main.py` | FastAPI app, middleware, routers, lifespan |
| `src/routers/` | `/v3/{resource}/{id}/{action}` HTTP surface (workspaces, peers, sessions, messages, conclusions, keys, webhooks) |
| `src/dialectic/` | The tool-using agent that answers chat queries (5 reasoning tiers) |
| `src/deriver/` | Background worker — queue manager, consumer, minimal deriver |
| `src/dreamer/` | Off-cycle consolidation — orchestrator, DeductionSpecialist, InductionSpecialist, surprisal sampling, reasoning trees |
| `src/reconciler/` | In-process scheduler for embedding sync + queue cleanup (hosted by deriver) |
| `src/llm/` | Provider-agnostic LLM layer — Anthropic, OpenAI, Gemini backends with fallback chains and `AttemptPlan` |
| `src/utils/agent_tools.py` | Unified tool registry; per-agent tool lists (2566 lines) |
| `src/utils/search.py` | Hybrid search — pgvector HNSW + Postgres FTS + RRF |
| `src/utils/summarizer.py` | Two-tier session summarization |
| `src/vector_store/` | Pluggable external vector stores (Turbopuffer, LanceDB) |
| `migrations/versions/` | Alembic migrations — the `(observer, observed)` rename (`08894082221a_replace_collection_name_*`) and the Peer Paradigm migration (`d429de0e5338_adopt_peer_paradigm.py`) are the most informative |
| `sdks/python/`, `sdks/typescript/` | Public SDKs (`honcho-ai` PyPI, `@honcho-ai/sdk` npm) |
| `mcp/` | Cloudflare Worker exposing the API as MCP tools |
| `honcho-cli/` | Python CLI for inspecting deployments (new in 3.0.7, `CHANGELOG.md:22`) |
| `examples/` | CrewAI, LangGraph, n8n, Gmail, Granola, Zo integrations |
| `docs/v3/` | Mintlify docs (api-reference, guides, openapi.json) |

If you only read one file: `src/models.py`. The `(observer, observed)` uniqueness constraint and the `Document.level` + `Document.source_ids` columns are the whole architectural bet, encoded.

## Mental model for using it well

- **Treat peers as relationships, not identities.** "What does Alice know about the coach?" is `observer=alice, observed=coach`. "What does the coach know about Alice?" is `observer=coach, observed=alice`. They are different collections with different conclusions. This is the design's whole point.
- **Don't expect immediate consistency on writes.** The deriver is async by construction. If you `add_messages` and immediately call `peer.chat`, the new messages may not yet be reflected in the representation (`README.md:154`). For low-latency reads use the `representation` endpoint, which returns the snapshot synchronously.
- **Use `observe_others=true` deliberately.** It's off by default. Flipping it on per-session is what unlocks "what does Alice's agent understand about the other participants in this session" — but it multiplies the number of collections and the deriver workload.
- **Lean on the Dreamer for consolidation.** The minimal deriver only extracts explicit facts. The deductive/inductive layers exist; they require dreams to fire. If you disable `DREAM.ENABLED` you get a flat explicit-only memory.
- **The Dialectic's reasoning tier is a cost dial.** `minimal` uses two tools and a cheap model; `max` uses seven tools and a top-tier model. The tier defaults at `low` (`src/dialectic/core.py:70`) and you can override per call.
- **Trust the composite FK scoping.** You cannot accidentally cross workspaces. The schema enforces it, not the application code (`CLAUDE.md:172-176`).

## When NOT to reach for this

- **You need synchronous write-then-read consistency.** Honcho's whole architecture is "API writes return immediately; reasoning is async." If your agent's correctness depends on the message it just wrote being visible to its next query, you'll fight the design.
- **You can't accept AGPL.** Network-server copyleft. If you ship a SaaS that uses a modified Honcho, you must release the modifications. The managed `api.honcho.dev` is the explicit escape hatch.
- **You only have one user and one agent and no theory-of-mind requirement.** The `(observer, observed)` machinery is overhead you don't need. [[Profile__Mem0]] with `user_id` scope is dramatically simpler.
- **You're a coding agent.** Honcho models people and relationships, not file trees and refactors. [[Profile__Volt]] is the right shape for that.
- **You need provenance and supersession in the schema.** Honcho carries `source_ids` for the reasoning chain but not a `superseded_by` pointer. State-correctness-over-time is delegated to the Dreamer's consolidation, not enforced structurally. See `[[Profile__Neo]]` and `[[Profile__StateBench]]`.
- **You want a pure embedded library.** Honcho is a stateful service. You need Postgres, Redis, and a worker process. There is no in-process mode the way Mem0 ships `pip install mem0ai` and runs against local SQLite.

## How this compares to the rest of the study

| Axis | Honcho | Mem0 | Letta | Volt | Neo |
|---|---|---|---|---|---|
| **Shape** | Multi-peer memory server | General memory layer | Stateful agent runtime | Coding agent + memory | Code-reasoning memory + agent |
| **Indexing axis** | `(observer, observed)` pair | user/agent/run/actor | actor (user/agent) | conversation | scope hierarchy |
| **Storage** | Postgres + pgvector + Redis | Vector + entity store + SQLite | Postgres + pgvector | Postgres DAG | Scoped JSON files |
| **Write policy** | Async deriver + scheduled dreamer | Single-pass extract + entity link | Function-call mutations | Verbatim append + async summary | Append + deterministic supersession |
| **Recall** | Agentic Dialectic over hybrid FTS+vector | Hybrid (semantic + BM25 + entity) | In-context + archival recall | Bindle expansion + `lcm_grep` | Semantic + confidence × success |
| **Reasoning depth** | 4 levels (explicit/deductive/inductive/contradiction) + reasoning trees | Flat | Flat | Flat | Flat with supersession |
| **Theory of mind** | First-class (`observe_others`) | Not modeled | Not modeled | Not modeled | Not modeled |
| **Eviction** | Soft delete + dream-driven consolidation | None in OSS | Manual | Bindle eviction | Stale-prune + demote |
| **License** | AGPL-3.0 | Apache-2.0 | Apache-2.0 | varies | varies |
| **Best fit** | Multi-participant agents, coaches, therapists, group dynamics | Per-user agent memory | Long-running stateful agents | Long-horizon coding sessions | Code assistants improving over time |

## How this compares to our own `context-vigilance` skill

Same dichotomy as [[Profile__Mem0]]: `context-vigilance` is a human-readable markdown discipline for project knowledge; Honcho is the wrong tool when the consumer is a human reading a PR and the right tool when the consumer is an agent that needs to model relationships between people over time across sessions.

The honest overlap: if you're building a Lossless agent that talks to multiple humans about each other (a deck-reviewer agent that knows what the CEO, the lead, and the analyst each think about the same slide), Honcho's `(observer, observed)` is exactly the shape you want, and there is **no equivalent in `context-vigilance`**.

## One-line summary

> Honcho is multi-peer memory infrastructure where every derived conclusion is keyed by `(observer, observed)`, the write path is an async deriver + a scheduled dreamer that produce a four-level reasoning tree, recall is a tool-using Dialectic agent over hybrid FTS+vector search, and the AGPL license + managed-API duality is the explicit business model — making it the right default whenever the memory question is "what does peer A understand about peer B over time" and the wrong default for everything else.
