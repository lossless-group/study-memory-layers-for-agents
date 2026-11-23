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
site_uuid: fe8ff9fd-8a94-4a50-9595-b75e118d6ff1
hex_code: nvfrxl
date_authored_initial_draft: 2026-05-26
date_authored_current_draft: 2026-11-23
lede: >-
  Collections are unique on (observer, observed, workspace) — Honcho indexes
  memory by relationship rather than by user.
summary: >-
  Profile of Honcho (Plastic Labs) as pinned in the memory-layers-for-agents
  study. It is the only entry that models theory of mind in the schema, and the
  right reference whenever the memory question involves multiple participants
  reasoning about each other. Covers the peer paradigm, the async write path
  (immediate enqueue, then a minimal single-LLM-call deriver worker, then a
  scheduled Dreamer running DeductionSpecialist and InductionSpecialist with
  surprisal-based prioritization), the four DocumentLevel values and the
  source_ids reasoning tree, hybrid pgvector + Postgres FTS recall fused by RRF,
  the tool-using Dialectic agent with its five cost tiers, and peer-presence
  filtering that only returns messages a peer was actually present for. Note two
  adoption constraints: no write-then-read consistency, and AGPL-3.0.
publish: true
title: Honcho — Profile
date_modified: 2026-11-23
authors:
  - Michael Staton
status: Reference
type: Profiles
---

# Honcho — Profile

A profile of Honcho as it lives in this study (`studies/memory-layers-for-agents/honcho/`). Read this alongside [[Profile__Mem0]] and [[Profile__Letta]] — the three define almost the whole "general-purpose agent memory" frontier. Honcho is the one that takes **theory-of-mind** seriously enough to put it in the schema.

## TL;DR

Honcho is **multi-peer memory infrastructure** built around an `(observer, observed)` keying of all derived knowledge. Storage is Postgres with `pgvector` (`pyproject.toml:15`, `database/init.sql:1`); the load-bearing tables are `messages` (immutable append-only, FTS-indexed) (`src/models.py:477-563`) and `collections` (`src/models.py:749-831`) — and every collection is **unique on `(tenant_id, observer, observed, workspace_name)`** (`src/models.py:787-793`), which is the single line of code that distinguishes Honcho from every other system in this study.

The write path is **append-only messages + async background reasoning**. Messages enqueue queue items (`src/deriver/enqueue.py:39-113`); a separate **deriver worker process** (`uv run python -m src.deriver`, `src/deriver/__main__.py:105-116`) consumes them and runs a **single structured-output LLM call per batch** (`src/deriver/deriver.py:45-195`) that emits explicit conclusions into `(observer, observed)` collections. A scheduled **Dreamer** (`src/dreamer/orchestrator.py:127-425`) wakes up off-peak and runs a two-phase **DeductionSpecialist → InductionSpecialist** consolidation pass that produces deductive and inductive conclusions, with surprisal-based prioritization (`src/dreamer/surprisal.py:46-60`). The API never blocks on any of this.

Recall is a **tool-using Dialectic agent** (`src/dialectic/core.py:81-180`) — the only synchronous LLM call on the request path. It loops over a tool set (`search_memory`, `search_messages`, `get_observation_context`, `grep_messages`, `get_messages_by_date_range`, `search_messages_temporal`, `get_reasoning_chain`) at one of five reasoning tiers (`minimal/low/medium/high/max`) until it has enough context to answer, then synthesizes natural-language output. Search itself is **hybrid** — Postgres full-text-search (GIN index on `to_tsvector('english', content)`, `src/models.py:559-563`) fused with HNSW cosine similarity on pgvector via Reciprocal Rank Fusion (`src/utils/search.py:49-90`).

If you want one sentence: **Honcho is a multi-peer Postgres+pgvector memory system where storage is keyed by `(observer, observed)`, ingest is async (a deriver worker plus a scheduled dreamer), and recall is a tool-using Dialectic agent over hybrid FTS+vector search — the only system in this study that natively models "what does peer A understand about peer B?"**

## Why this exists — the design bet

Most agent-memory systems index by user. Honcho indexes by *relationship*. The thesis (`CLAUDE.md:23-30`, "Peer Paradigm") is that "users" and "agents" are the same kind of object — **peers** — and the interesting unit of memory is not "what does the system know about Alice" but "what does Alice's representation-within-the-coach-agent look like, and how does that differ from the same coach's representation within Alice's session with the therapist?"

1. **`(observer, observed)` is the indexing axis, not a metadata field.** The `Collection` table is unique on `(tenant_id, observer, observed, workspace_name)` (`src/models.py:787-793`; `tenant_id` is `'default'` on a single-tenant install). Every derived conclusion (`Document`) lives in exactly one collection (`src/models.py:832-1085`). Self-representation is just the special case `observer == observed`.
2. **The write path is asynchronous on purpose.** Messages return from the API immediately (`src/deriver/enqueue.py:39-113`); reasoning happens in a separate worker process so the synchronous path never blocks on LLM latency. The README's own caveat — "newly-added messages may take a moment to be reflected in chat/representation responses" (`README.md:158`) — is the explicit cost.
3. **The deriver is deliberately "minimal."** A single structured-output LLM call per batch, not an agentic tool loop (`src/deriver/deriver.py:36-60`, `CLAUDE.md:166`). The agentic loop is reserved for two places where it earns its keep: the Dialectic on read (`src/dialectic/core.py:81-180`) and the Dreamer specialists off-cycle (`src/dreamer/specialists.py:838-1117`).
4. **Conclusions form a reasoning tree.** Each `Document` carries `source_ids` (premise documents) and a `level` ∈ `{explicit, deductive, inductive, contradiction}` (`src/models.py:845-851, 889-904`, `src/utils/types.py:312`). The Dialectic can traverse the chain via `get_reasoning_chain` (`src/utils/agent_tools.py:603`).
5. **Sessions are many-to-many with peers via a temporal membership table.** `session_peers` carries `joined_at` / `left_at` (`src/models.py:139-190`, `src/models.py:1420-1431`), and recall filters messages by whether a peer was *actually present* at the time the message was created (`src/utils/search.py:219-287`). This is what makes "what did Alice know at the time" answerable, not just "what's in the corpus now."

## Storage topology

| Layer | Backend | Purpose | Code |
|---|---|---|---|
| Primary OLTP | Postgres 15+ with `pgvector` extension | All resources (workspaces, peers, sessions, messages, collections, documents, queue) | `pyproject.toml:15`, `database/init.sql:1`, `docker-compose.yml.example:113-136` |
| Vector | `pgvector` HNSW indexes on `MessageEmbedding.embedding` and `Document.embedding` (m=16, ef_construction=64, cosine_ops) | Semantic search over messages and conclusions | `src/models.py:718-725`, `src/models.py:1046-1053` |
| Full-text | Postgres GIN on `to_tsvector('english', content)` | Keyword/BM25-like search over message content | `src/models.py:559-563` |
| Cache / async coordination | Redis (`cashews` client) | Hot-path caching, queue scheduling, session locks | `pyproject.toml:35-36`, `src/cache/client.py` |
| Pluggable external vector | Turbopuffer, LanceDB, Qdrant, or ChromaDB (optional) | Swap pgvector out for managed/embedded vector stores | `pyproject.toml:34,48-56`, `src/vector_store/__init__.py:205-230`, `src/vector_store/{turbopuffer,lancedb,qdrant,chroma}.py` |
| Queue | `queue` table in same Postgres | Background tasks (representation, summary, dream, webhook, deletion, reconciler) | `src/models.py:1189-1285` |
| MCP edge | Cloudflare Worker, plus stdio and Streamable HTTP hosts | Remote MCP server at `mcp.honcho.dev`, or a local/self-hosted MCP server | `mcp/README.md`, `mcp/wrangler.toml`, `mcp/src/{stdio,http}.ts` |

Note what's **not** in the stack: no separate graph DB, no Neo4j, no Elasticsearch, no separate object store. The whole system is "Postgres + Redis + a worker process." Since 3.6.0 the tenant-scoped tables are hash-partitioned by `tenant_id` with row-level security available behind `MULTI_TENANT` (`src/models.py:127-135`, `CLAUDE.md:120`); off by default, so a self-hosted install still runs single-tenant on plain Postgres. That is the operational selling point.

## Schema of a memory record

The two load-bearing tables, with the columns that matter for memory semantics.

**`collections`** — the `(observer, observed)` keying (`src/models.py:749-831`):

```python
class Collection(Base):
    id: str                  # nanoid
    observer: str            # peer name, indexed
    observed: str            # peer name, indexed
    workspace_name: str
    tenant_id: str           # added in 3.6.0; 'default' on single-tenant installs
    h_metadata: JSONB
    internal_metadata: JSONB
    # UNIQUE(tenant_id, observer, observed, workspace_name)
```

**`documents`** — the conclusion itself (`src/models.py:832-1085`):

```python
class Document(Base):
    id: str                       # nanoid
    content: str                  # the conclusion text
    level: DocumentLevel          # "explicit" | "deductive" | "inductive" | "contradiction"
    times_derived: int            # how many dream cycles have re-derived this
    embedding: Vector(_VECTOR_DIM)
    source_ids: list[str] | None  # premise document IDs (reasoning tree); now backed by the
                                  # document_sources edge table (src/models.py:1087)
    observer: str
    observed: str
    workspace_name: str
    session_name: str | None
    deleted_at: datetime | None   # soft delete
    sync_state: "synced|pending|failed"   # embedding pipeline state
    created_at: datetime
```

**`messages`** — raw, immutable, FTS-indexed (`src/models.py:477-563`):

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

- **`explicit`** — extracted directly from a message by the deriver (`src/deriver/prompts.py:46`).
- **`deductive`** — logical necessity inferred by the DeductionSpecialist during a dream (`src/dreamer/specialists.py:838`, `src/utils/agent_tools.py:729-743`).
- **`inductive`** — pattern across many explicit/deductive facts, produced by the InductionSpecialist (`src/utils/agent_tools.py:744-756`).
- **`contradiction`** — conflict flagged for resolution (`src/utils/types.py:240`).

The reasoning chain is reified in the data — `source_ids` is GIN-indexed for tree traversal (`src/models.py:1066-1080`), and since 3.5.0 explicit conclusions also cite the messages they came from through a `conclusion_source_messages` edge table (`src/models.py:1135`). [[Profile__Mem0]]'s entity-link array is the closest analog, but it doesn't carry level or premise semantics.

## Write policy — append-only messages + async deriver + scheduled dreamer

The write path has **three temporally separated stages**:

**1. API enqueue (synchronous, fast).** `POST /v3/.../messages` (or `add_messages` from the SDK) writes immutable rows into `messages` and inserts queue items into `queue` (`src/deriver/enqueue.py:39-113`). Returns immediately.

**2. Minimal deriver (async, per-batch).** A separate worker process (`uv run python -m src.deriver`, `src/deriver/__main__.py`) consumes `queue` items grouped by **work unit key** — `representation:{workspace}:{session}:{observed}`, prefixed by tenant when `MULTI_TENANT` is on (`src/utils/work_unit.py:48-90`). For each batch it makes **a single LLM call** with the `minimal_deriver_prompt` (`src/deriver/prompts.py:46`) and structured output, extracting explicit atomic facts about the `observed` peer. Each extracted observation becomes a `Document` row in every collection where some other peer observes this one (`src/deriver/deriver.py:45-195`). This is a deliberate trade — `CLAUDE.md:166` calls this "minimal deriver" and contrasts it with an agentic loop: "trades flexibility for cost and predictability."

**3. Dreamer (scheduled, off-peak).** A `DreamScheduler` (`src/dreamer/dream_scheduler.py:34`) accumulates pending dreams per `(observer, observed)` and fires them on a delay after activity stops (cancelled if the peer becomes active again, `src/deriver/enqueue.py:51-63`). A dream run (`src/dreamer/orchestrator.py:127-425`) is two specialist phases (a third, `CardRefreshSpecialist`, runs alone as a lightweight `card_refresh` dream that only updates the peer card, `src/dreamer/orchestrator.py:427`, `src/dreamer/specialists.py:1118`):
- **DeductionSpecialist** (`src/dreamer/specialists.py:838`, tools at `src/utils/agent_tools.py:729-743`): an agentic loop that creates deductive conclusions from explicit ones, can delete duplicates, and writes to the **peer card** (a small identity-marker list capped at 40 entries × 200 chars, `src/utils/agent_tools.py:49-61`).
- **InductionSpecialist** (`src/dreamer/specialists.py:985`, tools at `src/utils/agent_tools.py:744-756`): creates inductive patterns from explicit + deductive conclusions. Explicitly forbidden from writing the peer card (`CHANGELOG.md:476`, 3.0.7 entry). Since 3.6.0 each dream is recorded as a run with its changes (`DreamRun`, `DreamRunChange` at `src/models.py:1435-1531`), readable through `src/routers/dreams.py` as "Representation History".

Surprisal scoring (`src/dreamer/surprisal.py:46`) pre-filters which observations the specialists focus on — high-surprisal facts (geometrically distant from the existing tree) get prioritized.

The Summarizer (`src/utils/summarizer.py`, `CLAUDE.md:201-207`) runs alongside as a direct LLM call (no tools), producing two-tier session summaries: **short** every 20 messages, **long** every 60 messages.

**There is no supersession primitive in the schema.** A contradicted conclusion gets a sibling `contradiction`-level document rather than a `superseded_by` pointer (compare `[[Profile__Neo]]` §schema). The Dreamer can `delete_observations` (soft delete via `deleted_at`, `src/models.py:861-863`) but the API user can also `DELETE /v3/.../conclusions/{id}` directly (`src/routers/conclusions.py:166-171`). State-correctness over time is therefore the Dreamer's job, not the schema's.

## Recall API

Two distinct shapes, with very different cost profiles:

**Low-latency hybrid search** (`peer.search(...)`, `session.search(...)`, `honcho.search(...)`) — no LLM call. Pgvector HNSW + Postgres FTS, fused with Reciprocal Rank Fusion (`src/utils/search.py:49-90`):

```python
RRF_score = sum(1 / (k + rank_i))   # k=60 default
```

The vector search oversamples by 2× to handle deduplication across chunked embeddings without breaking the HNSW index scan (`src/utils/search.py:207`, and 4× under peer-perspective filtering at `:503`). FTS falls back to ILIKE when the query has special characters (`src/utils/search.py:373-430`; the fallback is backed by a `pg_trgm` GIN index since 3.3.0).

**Agentic Dialectic** (`peer.chat(query=...)`, `POST /v3/.../peers/{peer_id}/chat`) — the headline endpoint. A tool-using LLM agent (`src/dialectic/core.py:81-180`) runs a loop over the following tools (`src/utils/agent_tools.py:672-681`):

| Tool | What it does |
|---|---|
| `search_memory` | Vector search over `documents` in `(observer, observed)` collection |
| `search_messages` | Hybrid search over `messages` |
| `get_observation_context` | Fetch the original messages a conclusion came from |
| `grep_messages` | Exact-text ILIKE for names/dates/keywords |
| `get_messages_by_date_range` | Temporal slicing |
| `search_messages_temporal` | Semantic + date filtering combined |
| `get_reasoning_chain` | Traverse `source_ids` to walk the deductive tree |

The agent has **five reasoning tiers** (`minimal/low/medium/high/max`) each with its own model config and tool set — `minimal` reduces to just `search_memory` + `search_messages` (`src/utils/agent_tools.py:683-688`) for cost. The default returns natural language; streaming is supported (`src/dialectic/chat.py`). Since 3.1.0 there is also a workspace-level chat (`POST /v3/workspaces/{workspace_id}/chat`) that reads across every pair in the workspace with its own tool set (`src/utils/agent_tools.py:695-713`), and since 3.2.0 an opt-in `include_evidence` flag returns the conclusions and messages the agent read.

**Peer-perspective filtering** (`src/utils/search.py:219-287`) is the part most other systems can't do: messages from a session are only returned if the peer was a member at the time the message was created. Time-windowed visibility, structurally enforced.

## Eviction & compaction

There is **no automatic TTL**. The cleanup primitives that exist:

- **`Document.deleted_at`** (`src/models.py:861-863`) — soft delete. The Dreamer's `delete_observations` tool uses this (`src/utils/agent_tools.py:588`). Since 3.5.0 workspaces, peers, sessions and messages also carry `deleted_at` tombstones, and reads skip rows under a tombstoned parent.
- **Two-tier summarization** (`src/utils/summarizer.py`, `CLAUDE.md:201-207`) — short summary every 20 messages, long every 60. Reduces the working context size for the Dialectic without deleting source messages.
- **Reconciler** (`src/reconciler/`, hosted by the deriver worker, `CLAUDE.md:154`) — embeds `MessageEmbedding` rows with `sync_state='pending'` and removes stale queue items. This is a *consistency* loop, not a *forgetting* loop.
- **Dream-driven consolidation** (`src/dreamer/specialists.py`) — DeductionSpecialist can mark duplicates for deletion. This is the closest Honcho gets to active forgetting.
- **`times_derived`** counter on `Document` (`src/models.py:848-850`) — tracks how often a conclusion has been re-derived. Available to the Dreamer for relevance scoring but no automatic demotion policy.

Compare [[Profile__Volt]]'s deterministic three-level escalation or [[Profile__Neo]]'s `prune_stale_facts`. Honcho's bet is that consolidation (deduction collapses redundancy, induction abstracts) is enough — and that you should preserve message history rather than compress it.

## Scopes & namespacing

A four-level hierarchy enforced by composite foreign keys (`CLAUDE.md:331` "Composite-FK multi-tenancy"), with an optional tenant level above it since 3.6.0:

```
Tenant (optional, MULTI_TENANT)        # control-plane root, /v3/tenants
Workspace                              # tenant root on a single-tenant install
  └── Peer (uniq on name, workspace)   # human OR agent — same type
        └── Session (m2m via session_peers, temporal joined_at/left_at)
              └── Message (uniq on workspace, session, seq_in_session)
                    └── (after deriver) Document in Collection(observer, observed)
```

`workspace_name` participates in nearly every composite FK, and since 3.6.0 every key leads with `tenant_id` (`src/models.py:326, 373, 445, 787`), which makes cross-workspace data leakage structurally impossible at the schema level. This is stronger than [[Profile__Mem0]]'s metadata-filter scope and roughly equivalent to [[Profile__Letta]]'s actor-based namespacing.

The **peer observation switches** are per-session, per-peer (`src/schemas/configuration.py:289-305`):

- `observe_me` — whether Honcho should form *any* representation of this peer.
- `observe_others` — whether this peer forms session-level representations of other peers (i.e., creates `(this_peer, other_peer)` collections).

That `observe_others` flag is the explicit theory-of-mind switch. Off by default; flip it on per-session when you want a coach-agent that models how a user thinks about their therapist.

**Scopes** (3.1.0, `src/routers/scopes.py`) add a named grouping of sessions that bounds recall. A scope is implemented as an observer peer named `scope.{name}`, so it reuses the `(observer, observed)` machinery rather than adding a new axis; named-scope API keys (3.4.0) pin a caller to one scope.

## Operational story

Two cooperating processes that share Postgres + Redis (`CLAUDE.md:150-154`):

1. **API server** — `uv run fastapi dev src/main.py` (`src/main.py`). Handles HTTP, enqueues background work, hosts the **Dialectic** agent inline on `/chat`.
2. **Deriver worker** — `uv run python -m src.deriver` (`src/deriver/__main__.py:105-116`), uvloop-based. Long-running queue consumer running the Deriver, Summarizer, and Dreamer off `queue` rows. Scalable horizontally via `DERIVER_WORKERS`. Hosts an in-process `ReconcilerScheduler` (`src/reconciler/`).

The reference Docker stack (`docker-compose.yml.example`) is exactly four services: `api` (port 8000), `deriver`, `database` (pgvector/pgvector:pg15), `redis` (redis:8.2). Optional Prometheus + Grafana for observability. The compose file now also carries an `mcp` service. The whole thing runs locally with `docker compose up -d --build`, or with `honcho start --setup` from the CLI (`README.md:52`).

Three deployment modes, one API:

1. **Managed** — `api.honcho.dev`. SDK defaults here. `MemoryClient`-style usage via `Honcho(workspace_id=..., api_key=...)` (`README.md:93`).
2. **Self-hosted** — clone the repo, copy `docker-compose.yml.example`, `docker compose up`, or `honcho start` (`README.md:77`).
3. **MCP edge** — a Cloudflare Worker (`mcp/README.md:1-50`) exposes the whole API surface as MCP tools at `mcp.honcho.dev`. Self-hosters point it at their own deployment via `HONCHO_API_URL` (`CHANGELOG.md:461`). Since 3.1.2 the same MCP server also runs over stdio (`mcp/src/stdio.ts`) and Streamable HTTP (`mcp/src/http.ts`) for local clients.

Telemetry is CloudEvents-based (`src/telemetry/events/`, `CLAUDE.md:213`), with `LLMCallCompletedEvent` carrying full cost attribution per provider hit (`CHANGELOG.md:454`). Langfuse (v5 SDK) and Sentry are first-class (`pyproject.toml:16,26`).

**License is AGPL-3.0** (`LICENSE:1-2`). Network-server copyleft: any modified version exposed over a network must publish source. The managed `api.honcho.dev` exists in part to give commercial users a way to consume Honcho without taking on AGPL obligations — this is the explicit business model. Compare [[Profile__Mem0]] (Apache-2.0, no copyleft) and [[Profile__Letta]].

## What's inside this submodule

| Path | What's there |
|---|---|
| `src/models.py` | SQLAlchemy ORM — the load-bearing schema (1,877 lines; tenancy, dream runs and ingest tables account for most of the growth) |
| `src/main.py` | FastAPI app, middleware, routers, lifespan |
| `src/routers/` | `/v3/{resource}/{id}/{action}` HTTP surface (workspaces, peers, sessions, messages, conclusions, keys, webhooks, scopes, dreams, ingest, tenants, deriver metrics) |
| `src/dialectic/` | The tool-using agent that answers chat queries (5 reasoning tiers) |
| `src/deriver/` | Background worker — queue manager, consumer, minimal deriver |
| `src/dreamer/` | Off-cycle consolidation — orchestrator, DeductionSpecialist, InductionSpecialist, CardRefreshSpecialist, surprisal sampling, reasoning trees |
| `src/reconciler/` | In-process scheduler for embedding sync + queue cleanup (hosted by deriver) |
| `src/llm/` | Provider-agnostic LLM layer — Anthropic, OpenAI, Gemini backends with fallback chains and `AttemptPlan` |
| `src/utils/agent_tools.py` | Unified tool registry; per-agent tool lists (2,990 lines) |
| `src/utils/search.py` | Hybrid search — pgvector HNSW + Postgres FTS + RRF |
| `src/utils/summarizer.py` | Two-tier session summarization |
| `src/vector_store/` | Pluggable external vector stores (Turbopuffer, LanceDB, Qdrant, ChromaDB) and per-tenant namespaces (`tenant_namespace.py`) |
| `migrations/versions/` | Alembic migrations — the `(observer, observed)` rename (`08894082221a_replace_collection_name_*`) and the Peer Paradigm migration (`d429de0e5338_adopt_peer_paradigm.py`) are the most informative |
| `sdks/python/`, `sdks/typescript/` | Public SDKs (`honcho-ai` PyPI, `@honcho-ai/sdk` npm) |
| `mcp/` | MCP server: Cloudflare Worker, stdio and Streamable HTTP hosts |
| `honcho-cli/` | Python CLI for inspecting deployments (new in 3.0.7, `CHANGELOG.md:460`); now also `honcho start` for a local stack and `honcho install` to wire Honcho into local AI agents |
| `connectors/`, `src/routers/ingest.py` | Ingest API (unreleased at this pin) for importing source records, with Discord and GitHub connectors |
| `hermes-plugin-honcho/`, `harness-plugin-core/`, `kilo-honcho/`, `integrations/` | Agent-harness plugins (Hermes, Cursor/Grok Bot, Kilo, and others) |
| `examples/` | CrewAI, LangGraph, n8n, Gmail, Granola, Zo integrations |
| `docs/v3/` | Mintlify docs (api-reference, guides, openapi.json) |

If you only read one file: `src/models.py`. The `(observer, observed)` uniqueness constraint and the `Document.level` + `Document.source_ids` columns are the whole architectural bet, encoded.

## Mental model for using it well

- **Treat peers as relationships, not identities.** "What does Alice know about the coach?" is `observer=alice, observed=coach`. "What does the coach know about Alice?" is `observer=coach, observed=alice`. They are different collections with different conclusions. This is the design's whole point.
- **Don't expect immediate consistency on writes.** The deriver is async by construction. If you `add_messages` and immediately call `peer.chat`, the new messages may not yet be reflected in the representation (`README.md:158`). For low-latency reads use the `representation` endpoint, which returns the snapshot synchronously.
- **Use `observe_others=true` deliberately.** It's off by default. Flipping it on per-session is what unlocks "what does Alice's agent understand about the other participants in this session" — but it multiplies the number of collections and the deriver workload.
- **Lean on the Dreamer for consolidation.** The minimal deriver only extracts explicit facts. The deductive/inductive layers exist; they require dreams to fire. If you disable `DREAM.ENABLED` you get a flat explicit-only memory.
- **The Dialectic's reasoning tier is a cost dial.** `minimal` uses two tools and a cheap model; `max` uses seven tools and a top-tier model. The tier defaults at `low` (`src/dialectic/core.py:99`) and you can override per call.
- **Trust the composite FK scoping.** You cannot accidentally cross workspaces. The schema enforces it, not the application code (`CLAUDE.md:331`).

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

## Changes since the previous pin (2026-05-21 → 2026-11-21)

Pin moved from `7470866d1284` to `dcd320347313`: 432 commits, server releases 3.0.8 through 3.6.2 (`CHANGELOG.md:14-445`). The `(observer, observed)` thesis is intact; most of the work went into multi-tenancy, scoping, provenance and deletion. License is still AGPL-3.0.

- **Multi-tenancy (3.6.0).** Every tenant-scoped table gained `tenant_id`, leads its keys with it, and is hash-partitioned by it, with row-level security behind `MULTI_TENANT` (default off) and a control-plane registry at `/v3/tenants` (`src/models.py:127-135`, `src/models.py:221`, `src/routers/tenants.py`, `CLAUDE.md:120-121`). The collection uniqueness constraint is now `(tenant_id, observer, observed, workspace_name)` (`src/models.py:787-793`). The deriver claims work round-robin across tenants from a trigger-maintained `queue_item_batches` table (`src/models.py:1302`).
- **Scopes (3.1.0).** A named grouping of sessions that bounds recall, built as an observer peer `scope.{name}` (`src/routers/scopes.py`). Named-scope API keys followed in 3.4.0.
- **Workspace-level chat (3.1.0)** at `POST /v3/workspaces/{workspace_id}/chat`, a Dialectic that reads across all pairs (`src/utils/agent_tools.py:695-713`). `include_evidence` (3.2.0) returns what the agent read.
- **Message provenance for conclusions (3.5.0).** Explicit conclusions cite their source messages through `conclusion_source_messages` (`src/models.py:1135`); the reasoning tree moved to a `document_sources` edge table (`src/models.py:1087`).
- **Representation History (3.6.0).** Each dream is recorded as a `DreamRun` with per-conclusion `DreamRunChange` rows, readable through `src/routers/dreams.py` (`src/models.py:1435-1531`). An unreleased change keeps this history forever by default.
- **Dreamer changes.** A peer-card-only `card_refresh` dream with its own `CardRefreshSpecialist` (3.0.12, `src/dreamer/specialists.py:1118`); opt-in working memory (tool budget, brief, notes carried between dreams) in 3.4.0 (`CHANGELOG.md:120`).
- **Deletion.** Workspaces, peers, sessions and messages gained `deleted_at` tombstones (3.5.0), and reads skip anything under a tombstoned parent (3.6.0).
- **Ingest (unreleased).** `POST /v3/workspaces/{id}/ingest` imports source records idempotently, with Discord and GitHub connectors and a "derive nothing about bots" rule (`src/routers/ingest.py`, `src/models.py:1592`, `connectors/`).
- **Backends and surfaces.** Qdrant (3.2.0) and ChromaDB (3.3.0) vector stores; MCP over stdio and Streamable HTTP (3.1.2); `honcho start` and `honcho install` in the CLI; harness plugins for Hermes, Cursor/Grok Bot and Kilo; a `pg_trgm` index behind the ILIKE fallback (3.3.0). The server now requires Python 3.13 (3.1.1).
- **Docs.** "Abduction" was dropped from the documented reasoning capabilities (#1489); the four `DocumentLevel` values are unchanged (`src/utils/types.py:312`).

## One-line summary

> Honcho is multi-peer memory infrastructure where every derived conclusion is keyed by `(observer, observed)`, the write path is an async deriver + a scheduled dreamer that produce a four-level reasoning tree, recall is a tool-using Dialectic agent over hybrid FTS+vector search, and the AGPL license + managed-API duality is the explicit business model — making it the right default whenever the memory question is "what does peer A understand about peer B over time" and the wrong default for everything else.
