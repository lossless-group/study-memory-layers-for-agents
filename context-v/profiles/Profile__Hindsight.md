---
name: Hindsight Profile
slug: hindsight
upstream: https://github.com/vectorize-io/hindsight
package: hindsight-client (PyPI), @vectorize-io/hindsight-client (npm), hindsight-all (PyPI, embedded), hindsight-cli (cargo)
license: MIT
maintainer: Vectorize.io (Chris Latimer et al.)
study: studies/memory-layers-for-agents
profile_path: studies/memory-layers-for-agents/hindsight
profile_kind: full-stack (FastAPI server + Python/TS/Rust/Go SDKs + Rust CLI + Next.js control plane + Helm + Docker + embedded distribution)
date_created: 2026-05-26
site_uuid: e7c41ded-a09e-4fcc-9a7f-027d7b64ff44
hex_code: 6d829z
date_authored_initial_draft: 2026-05-26
date_authored_current_draft: 2026-11-23
lede: >-
  `fact_type` is a database CHECK constraint: world facts and first-person
  experiences travel through different extraction prompts.
summary: >-
  Profile of Hindsight (Vectorize.io) as pinned in the memory-layers-for-agents
  study. It is the study's largest single repo by surface area and its clearest
  example of a typed memory taxonomy. Covers the world/experience/observation
  taxonomy plus the mental-models layer, the deliberate everything-in-one-
  Postgres decision (pgvector + tsvector + JSONB + recursive CTEs), the
  multi-stage retain pipeline, 4-way parallel recall fused by RRF with
  cross-encoder reranking, the reflect tool hierarchy and its citation-enforcing
  done schema, banks as the isolation primitive, and consolidation as the
  eviction-and-supersession substitute. Note the benchmark posture: still
  vendor-self-reported, but with the cleanest third-party reproduction trail in
  the study.
publish: true
title: Hindsight — Profile
date_modified: 2026-11-23
authors:
  - Michael Staton
status: Reference
type: Profiles
---

# Hindsight — Profile

A profile of Hindsight as it lives in this study (`studies/memory-layers-for-agents/hindsight/`). Cites pinned paths so you can jump to source rather than trust paraphrase. Read alongside [`Profile__Mem0.md`](./Profile__Mem0.md) (same hybrid-retrieval thesis, different write axis), [`Profile__Neo.md`](./Profile__Neo.md) (single-fact-type with `superseded_by`), [`Profile__MemPalace.md`](./Profile__MemPalace.md) (no extraction, opposite bet), and [`Profile__StateBench.md`](./Profile__StateBench.md) (the harness Hindsight's numbers should eventually face).

## TL;DR

Hindsight is Vectorize.io's **agent memory system that ships as a full self-contained stack** — FastAPI server, Next.js control-plane UI, generated SDKs in four languages, a Rust CLI, a Helm chart, and an embedded "no server" Python distribution. It is the busiest single repo in the study by surface area: ~35 top-level directories and an eight-member uv workspace (`pyproject.toml:2`, see "What's inside this submodule" below) all coordinated around a three-verb API.

The architectural bet is a **typed memory taxonomy**. Every memory is classified at write time as one of three first-class types — *world fact* (general knowledge about the world), *experience* (the agent's first-person actions / observations), or *observation* (auto-consolidated synthesis of multiple memories) — with *mental models* (user-curated reflections) sitting in a separate table on top. The taxonomy is enforced in the database as a `CHECK` constraint: `fact_type IN ('world', 'experience', 'observation')` (`hindsight-api-slim/hindsight_api/models.py:177`), and the LLM extraction prompts are explicitly typed to emit one or the other (`hindsight-api-slim/hindsight_api/engine/retain/fact_extraction.py:180`, `:287-289`, `:1115-1117`).

The recall API is a **three-verb surface**: `retain` ingests, `recall` retrieves, **`reflect` analyzes the agent's own existing memories** to generate new observations and insights — i.e. introspective LLM reasoning over the bank's own contents (`hindsight-api-slim/hindsight_api/engine/reflect/agent.py:1-8`, `README.md:332-342`). That self-introspection move is the unusual one — most memory layers stop at "store and recall"; Hindsight makes "the agent thinks about what it knows" a first-class operation with its own agentic tool loop.

Storage is **deliberately polyglot inside one Postgres instance** (`hindsight-docs/docs/developer/storage.md:7-18`): pgvector with HNSW for semantics, native `tsvector` + GIN for BM25 lexical search, recursive CTEs for graph walks across `memory_links`, and timestamp columns + dedicated indexes for temporal filtering. Per-user / per-agent isolation lives in a **"bank"** abstraction (`hindsight-api-slim/hindsight_api/models.py:338-351`) — each bank is an isolated memory store with its own disposition traits (`skepticism`, `literalism`, `empathy`, 1–5).

Self-reported state-of-the-art on LongMemEval as of January 2026 (`README.md:44-52`; live results now at benchmarks.hindsight.vectorize.io), with claimed independent reproduction by the **Virginia Tech Sanghani Center** and **The Washington Post** — the cleanest provenance trail of any system in this study, though still not [[Profile__StateBench]]-style harness verification.

If you want one sentence: **Hindsight is a typed-taxonomy agent memory system (world / experience / observation / mental-model) with a three-verb API (`retain` / `recall` / `reflect`), 4-way parallel retrieval (semantic + BM25 + graph + temporal) fused by RRF and cross-encoder reranking, all on a single Postgres instance, shipped as a full open-source stack including Helm chart, Docker, embedded distribution, and SDKs in Python / TS / Rust / Go.**

## Why this exists — the design bet

Most agent memory prior to Hindsight either (a) treated memory as one undifferentiated type ranked at retrieval, or (b) ran a knowledge graph on the side and bolted it onto vector search. Hindsight's bet is that **the *type* of memory should drive the write, not just the retrieval ranking** — and that the agent should reason about its own memory store as a first-class operation. Five numbered choices:

1. **Memory has a taxonomy and the taxonomy is enforced.** `fact_type` is a `CHECK` constraint at the DB level, not a soft metadata field (`models.py:177`). World facts and experiences travel through structurally different extraction prompts (`fact_extraction.py:1115-1117, 1343-1344`) and are indexed separately (`models.py:182-197`, partial HNSW per fact_type). Compare to Mem0 (`Profile__Mem0.md` §schema) which carries `memory_type` as optional payload metadata, or to Neo (`Profile__Neo.md`) whose single fact type uses `superseded_by` to model change — Hindsight cuts the cake on a different axis.
2. **`reflect` is introspection, not retrieval.** It runs an agentic tool loop over the bank's own contents: `search_mental_models` → `read_mental_models` → `search_observations` → `recall` → `expand` → `done` (`reflect/tools_schema.py:18-262`). Since 1.2.0 the tool calls are made as replies to a response schema rather than through provider-native tool calling (`reflect/agent.py:2`). The hierarchy is deliberate ("search mental models FIRST", `:59`) — recall raw facts only when synthesized layers are stale or absent.
3. **Banks have personality.** Each bank carries `disposition` JSONB with `skepticism`, `literalism`, `empathy` traits (`models.py:338-349`) that are injected into the reflect prompt (`reflect/prompts.py:180-200`, which now spells out what each trait level means). Recall is unaffected; disposition only steers how the agent talks about what it found.
4. **One Postgres, all four retrieval strategies.** Vector / BM25 / graph / temporal all run in parallel against the same database (`engine/search/retrieval.py:1-8`, SQL in `engine/memories/pg/recall.py`). The "no storage abstraction" decision is explicit and documented (`hindsight-docs/docs/developer/storage.md:27-44`): they reject the multi-store fan-out other systems lean on (Mem0's Qdrant + SQLite + optional Neo4j; Neo's JSON files; MemPalace's filesystem) and accept the lock-in. One qualification since July 2026: the memory tables now sit behind a `MemoriesExtension` interface (`engine/memories/base.py:1-30`), so a third-party store can own a bank's memories while banks, operations, and mental models stay in Postgres. The default `PostgresMemories` keeps everything in Postgres, and the docs page still says "no storage abstraction".
5. **Ship the whole stack, not just the library.** Where Mem0 ships a SDK + optional server, Hindsight ships SDKs in four languages **generated from one OpenAPI spec** (`hindsight-clients/{python,typescript,rust,go}/`), a Rust CLI, a Next.js admin UI, a Helm chart for Kubernetes, a multi-arch Docker image, and `hindsight-all` — an embedded "no server" Python distribution that boots `pg0` (their forked single-binary Postgres) inside the process (`README.md:175-201`, `hindsight-docs/docs/developer/storage.md:49-67`).

## Storage topology

All on a single Postgres instance (pgvector + tsvector + JSONB + recursive CTEs). Oracle 23ai is supported with full feature parity via a dialect dispatcher (`CLAUDE.md:158-245`).

| Layer | Backend / Table | Purpose | Code |
|---|---|---|---|
| Raw memories | `memory_units` (Postgres + pgvector) | Sentence-level memories with embedding, `fact_type`, temporal columns | `models.py:132-204` |
| Per-fact-type HNSW | Partial indexes on `embedding` | Approx vector search, separate index per fact_type | `models.py:198-203`, `engine/memories/pg/graph.py:814` (per-transaction `hnsw.ef_search`) |
| BM25 | `tsvector` + GIN | Keyword/full-text retrieval | (Postgres native; `engine/memories/pg/recall.py`) |
| Entities | `entities` + `unit_entities` + `entity_cooccurrences` | Resolved canonical entities, M:N to memories, materialized cooccurrence cache | `models.py:207-290` |
| Graph edges | `memory_links` with typed `link_type` | `temporal / semantic / entity / causes / caused_by / enables / prevents`, weighted 0.0–1.0 | `models.py:293-335` |
| Source text | `documents` | Original text + `content_hash` for dedup | `models.py:111-129` |
| Observations | `memory_units` rows with `fact_type='observation'` | Auto-consolidated synthesis with `proof_count`, `source_memory_ids`, `history` | `engine/consolidation/consolidator.py:1-16, 791-850` |
| Mental models | `mental_models` table | User-curated reflections; `subtype IN ('structural','emergent','pinned','learned')` | `alembic/versions/h3c4d5e6f7g8_mental_models_v4.py:70-88`; refreshed pages are surfaced as **Pages** since 1.2.1 |
| Banks | `banks` | Isolation boundary; disposition JSONB; mission text | `models.py:338-351` |
| Curation archive | `invalidated_memory_units` | Facts a user invalidated through `PATCH /memories/{id}`, kept for audit and restore | `engine/memories/pg/curation.py:1-16` |
| Attachments | `attachments` | Images and files retained inline with a document (0.10.0) | `engine/retain/attachment_store.py` |
| Object storage | `engine/storage/{s3,gcs,azure,postgresql}.py` | Pluggable blob backend for chunk/document raw text at scale | `engine/storage/base.py` |

The "everything in Postgres" load-bearing decision is reasoned about explicitly: ACID across types, one connection string, one backup, one upgrade path (`hindsight-docs/docs/developer/storage.md:15-44`). The trade-off: you can't swap in a dedicated graph DB, and very large-scale graph queries hit the same instance as your hot vector path.

## Schema of a memory record

The `memory_units` row (`models.py:132-204`):

```python
{
    "id":              "uuid",                              # gen_random_uuid()
    "bank_id":         "...",                               # isolation key
    "document_id":     "...",                               # FK to documents
    "text":            "memory text",
    "embedding":       Vector(EMBEDDING_DIMENSION),         # pgvector; 384-dim default (config.py:1509)
    "context":         "...",                               # optional contextual hint at retain time
    "fact_type":       "world" | "experience" | "observation",  # CHECK constraint, models.py:177
    "event_date":      timestamp_tz,                        # required; back-compat
    "occurred_start":  timestamp_tz | None,                 # event range start
    "occurred_end":    timestamp_tz | None,                 # event range end
    "mentioned_at":    timestamp_tz | None,                 # when fact was mentioned
    "metadata":        JSONB,                               # user-defined str→str
    "created_at": "...", "updated_at": "..."
}
```

The extraction LLM emits an `ExtractedFact` (`fact_extraction.py:270-311`) with `what / when / where / who / why`, a `fact_type` (`world` or `assistant`, where `assistant` later maps to `experience` — see the deliberate translation at `fact_extraction.py:2304-2313`), and a `causal_relations` list with index-based references that **can only point backward** to previous facts (`:252-268`) — a hard guard against LLM-hallucinated indices.

The disposition payload on a bank (`models.py:344-346`):

```python
{"skepticism": 3, "literalism": 3, "empathy": 3}   # default; 1–5 each
```

Notable for what's **present**: the `memory_links.link_type` enum admits **causal** edges (`causes / caused_by / enables / prevents`, `models.py:320-322`), which neither Mem0 nor Neo carry as first-class types. Notable for what's **missing**: no `superseded_by` pointer (compare [[Profile__Neo]]) and no confidence field. Supersession is handled either by recall ranking or by the consolidation engine rewriting observations as new evidence arrives. Since 0.8.2 there is a manual path too: curation can edit or invalidate a world/experience fact, which moves it to an `invalidated_memory_units` archive table so recall no longer returns it but it can be restored (`hindsight-docs/docs/developer/api/memories.mdx:101-125`).

## Write policy — `retain`

The retain pipeline is multi-stage and explicit (`engine/retain/orchestrator.py`): chunk → extract facts → embed → resolve entities → store memory units → create links → enqueue consolidation. Focused modules in `engine/retain/` (`bank_utils`, `chunk_storage`, `embedding_processing`, `entity_processing`, `fact_extraction`, `fact_storage`, `link_creation`, plus newer `attachment_content`, `attachment_store`, `embedding_coalescer`, `entity_labels`, `fold`, `memory_budget`) keep each phase swappable.

Fact extraction is **LLM-typed at the prompt level**. The prompt explicitly tells the model how to choose between `world` and `assistant`/experience (`fact_extraction.py:1115-1117`):

> `"world"`: Objective/external facts, including the user's preferences, rules, corrections, constraints, plans, traits, or context. These stay "world" even when the user states them during an assistant interaction …
> `"assistant"`: Actions, experiences, or observations the assistant/agent actually performed … not merely for user facts mentioned in conversation.

At the previous pin the `assistant` bucket also covered "interactions with the user (requests, recommendations)". The current wording moves user preferences and corrections firmly into `world`, which narrows `experience` to what the agent itself did.

Three extraction modes ship: standard (`ExtractedFact`, `:270-311`), verbose (`ExtractedFactVerbose`, `:377-469`), and no-causal (`ExtractedFactNoCausal`, `:476-519`), selectable per bank via `retain_extraction_mode` config (`api/http.py:2194`). Per-bank `retain_mission` lets the operator steer what gets extracted (`api/http.py:2190`).

Hash-based dedup runs at the chunk level — `documents.content_hash` (`models.py:119`) is the dedup key.

After retain, an async consolidation job kicks off: it scans new memories scoped by tag, asks the LLM to either create a fresh observation or update/merge an existing one, and writes the result back to `memory_units` with `fact_type='observation'` carrying `proof_count`, `source_memory_ids`, and a `history` of changes (`consolidator.py:1-16`; since 0.8.0 that history lives in dedicated tables rather than inline JSONB). **This is the load-bearing eviction-substitute**: observations are not deleted, but they are continually rewritten as evidence accumulates, and stale-observation sweeps fire when source memories get deleted (`consolidator.py:706-760`).

## Recall API — 4-way parallel retrieval

```python
client.recall(bank_id="my-bank", query="What does Alice do?")
```

Four retrievers run **in parallel** for each requested fact type (`retrieval.py:129+`, which hands the arms to the memories store's `recall_unified`; Postgres SQL in `engine/memories/pg/recall.py`):

1. **Semantic** — pgvector HNSW similarity, with `hnsw.ef_search` set per transaction and per-fact-type partial indexes (`engine/memories/pg/graph.py:814`).
2. **BM25** — Postgres `tsvector` keyword match.
3. **Graph** — pluggable `GraphRetriever`; default walks `memory_links` (`retrieval.py:66-97`, `engine/search/graph_retrieval.py`, `engine/memories/pg/link_expansion.py`). Since 0.9.0 graph expansion, temporal search, and reranking can each be switched off per bank.
4. **Temporal** — optional time-range filtering, surfaced when the query carries dated intent (`engine/search/temporal_extraction.py`).

Results are merged via **Reciprocal Rank Fusion** (`engine/search/fusion.py:29-111`) — score is `Σ 1/(k + rank_in_source)` with `k=60`, the standard RRF constant — then re-scored by a **cross-encoder** with multiplicative recency / temporal / proof-count boosts (`engine/search/reranking.py:37-45`; recency decay is configurable as linear, exponential, or none since 0.8.4). Each boost contributes at most ±α/2 to the base CE score; defaults are α=0.2 for recency and temporal, α=0.1 for proof count.

The HTTP surface is at `POST /v1/default/banks/{bank_id}/memories/recall` (`api/http.py:5569-5580`). Output is shaped by `RecallRequest`'s `types`, `budget`, `max_tokens`, `query_timestamp`, `include` (entities / chunks / source_facts), `tags`, `tags_match`, and `tag_groups`. If `types` is omitted, all three fact types, observations included, are recalled (`api/http.py:5578, 5617`; `engine/response_models.py:22`). At the previous pin the default was `['world','experience']`. A `prefer_observations` option (0.8.4, with bank and server defaults since 1.1.0) returns observations in place of raw facts they have superseded.

## Reflect — the unusual verb

```python
client.reflect(bank_id="my-bank", query="What should I know about Alice?")
```

Reflect runs an **agentic LLM loop over the bank's own contents** (`engine/reflect/agent.py:1-8`). The agent gets a typed tool surface (`engine/reflect/tools_schema.py:18-262`):

| Tool | Purpose | When to use |
|---|---|---|
| `search_mental_models` | Curated reflections (highest quality) | FIRST when available (`:59`) |
| `read_mental_models` | Read a mental model in full by id, after search showed its snippet | Only for snippets that look relevant (`:117-148`) |
| `search_observations` | Auto-consolidated synthesis with freshness signal | Second; if stale, verify with recall |
| `recall` | Raw world/experience facts | Ground-truth fallback |
| `expand` | Get the source chunk or document for a memory | When the agent needs context the fact lost |
| `done` | Emit final answer + cited `memory_ids` / `mental_model_ids` / `observation_ids` + `answer_found` | Terminal |
| `answer_ready` | Stop signal for reflects whose answer is written in a closing step | Before `done` in that mode (`:380-404`) |

`done` enforces citations as schema (`tools_schema.py:216-261`): answers carry the IDs they were built from. It also requires an `answer_found` boolean; when that is false and recall has not run yet, the loop refuses the `done` and forces a recall first (`:198-214`). When the bank has **directives** (a flavor of mental model with `subtype='directive'`), the `done` tool gets rewritten to require a `directive_compliance` confirmation field (`tools_schema.py:319-378`) — agents can't ship an answer without explicitly attesting they obeyed each rule.

Disposition steers tone in the final-answer prompt (`reflect/prompts.py:180-200, 857-861`). The recall pipeline itself is disposition-blind (CLAUDE.md:317: "Disposition traits only affect reflect, not recall").

The HTTP surface is at `POST /v1/default/banks/{bank_id}/reflect` (`api/http.py:5816-5830`). Trace data (every tool call, every LLM call, durations) is returned when `request.include.tool_calls` is set (`api/http.py:5931-5951`).

That structural choice — **memory introspection as a tool-loop, not a single LLM pass** — is what differentiates `reflect` from a Mem0-style "search + LLM call on top." Compare to [[Profile__Mem0]] which has no equivalent verb; the closest analog is calling `search()` then handing results to your own agent.

## Eviction, consolidation, supersession

Hindsight has no TTL. The eviction substitute is the **consolidation engine** (`engine/consolidation/consolidator.py`), which runs as a background job after retain. Three primitive actions on observations (`consolidator.py:791-850`, `_CreateAction` / `_UpdateAction` / `_DeleteAction`):

- **Create** a new observation from novel facts.
- **Update** an existing observation when new evidence supports / contradicts / refines it.
- **Delete** an observation when its sources are gone (the "stale-observation sweep", `:706-760`).

Source memories are not rewritten by the pipeline. Since 0.8.2 an operator can curate them: `PATCH /v1/default/banks/{bank}/memories/{id}` edits a fact (re-embeds it and re-consolidates what was derived from it) or invalidates it into the `invalidated_memory_units` archive, restorable later (`hindsight-docs/docs/developer/api/memories.mdx:101-125`). Deletes and edits race safely against consolidation because `_filter_live_source_memories` takes `FOR SHARE` row locks (`consolidator.py:706-760`). Since 0.8.0, consolidation also deduplicates near-duplicate observations semantically by default (not on Oracle).

Supersession is **not modeled as a pointer chain** (compare [[Profile__Neo]]'s `superseded_by`). Instead, the assumption is: contradictory new facts → consolidation rewrites the observation → reflect prefers fresh observations → the old fact stays in the row but the synthesis layer above it has moved on. This is a deliberately optimistic stance: it's lighter-weight than explicit supersession, but it inherits the "two true-looking facts in the store" problem Mem0 has, with an extra layer that may smooth it over.

Mental models support refresh, dry-run refresh, and clear endpoints (`api/http.py:6587, 6629, 6679`), plus cron and delta refreshes, giving operators a manual override on the synthesis layer when the auto path drifts.

## Banks — the isolation primitive

A bank is "like a brain for one user/agent" (`CLAUDE.md:308-313`; the README now says "one 'brain' for one user, agent, or project"). The contract:

- **One bank per HTTP request.** All endpoints are scoped under `/v1/default/banks/{bank_id}/...` (`api/http.py:5570, 5817`). Multi-bank fan-out is the client's job.
- **Strict isolation.** `bank_id` is part of every table's index and most FKs (`models.py:127, 171-176, 178-197, 239-242`). No cross-bank queries from the engine.
- **Per-bank vector indexes (optional).** When `uses_per_bank_vector_indexes` is true, each (bank, fact_type) gets its own partial HNSW index (`engine/retain/bank_utils.py:53-56`, `_vector_index.py`) — useful for tenants with very different vector distributions.
- **Disposition + mission.** JSONB disposition steers reflect tone; a `mission` text field steers retain extraction (`api/http.py:2190`).
- **Per-bank config.** LLM provider, model, retain mode, chunk size, consolidation toggle — all overridable per bank via the configurable-fields registry (`CLAUDE.md:417-487`, `config.py` `_CONFIGURABLE_FIELDS`).

Since 0.10.2 banks can carry aliases for zero-downtime ID migrations, and since 0.10.1 a bank can be cloned or renamed in one call; 1.0.0 made tag-scoped callers unable to run bank-wide operations.

The bank model is more opinionated than Mem0's `user_id/agent_id/run_id` tuple — it forces operators to think about "what is the unit of isolation" up front, but trades that for richer per-tenant configuration.

## Operational story

Five deployment shapes shipped from one repo:

1. **Single-container Docker** (`docker run ghcr.io/vectorize-io/hindsight:latest`, `README.md:70-83`). Bundles API + UI + embedded pg0 Postgres on ports 8888 + 9999.
2. **Docker Compose with external Postgres** (`docker/docker-compose/`, `README.md:86-94`).
3. **Helm chart for Kubernetes** (`helm/hindsight/`). Multi-tenant, production-shaped.
4. **Embedded "no server" Python** (`pip install hindsight-all`, `README.md:175-201`). `HindsightServer` context manager boots pg0 in-process; same `HindsightClient` API hits a `localhost` URL the manager owns.
5. **Hindsight Cloud** (managed, usage-based, `README.md:117-119`).

LLM providers: the README now claims 25+ via `HINDSIGHT_API_LLM_PROVIDER` (`README.md:84`, `config.py:169`), including subscription-backed `openai-codex`, `claude-code`, `cursor`, and `github-copilot` that need no API key, plus local `ollama`, `lmstudio`, `llamacpp`. Embeddings default to 384-dim vectors (`config.py:1509`); local ONNX (0.8.0), TEI, NVIDIA NIM, and the vendor-hosted `hs-embed-1` are options. Since 1.2.0 an optional **decision model** (`hs-decide-1-flash`, 0.8B, Apache-2.0, with `hs-rerank-1` and `hs-embed-1`) handles small typed judgments inside recall, consolidation, reflect, and pages so fewer go to the main LLM (`README.md:364-381`).

Database backends: PostgreSQL (default, with pgvector / vchord / pgvectorscale / scann selectable via `DEFAULT_VECTOR_EXTENSION`, `config.py:1828`) or Oracle 23ai (`config.py:3613`). The dialect dispatcher (`alembic/_dialect.py` `run_for_dialect`, see `CLAUDE.md:163-245`) is enforced by CI lint — migrations that don't handle both dialects fail the build.

## Headline benchmark numbers

Source: `README.md:44-52` and `hindsight-docs/static/img/hindsight-benchmarks.png` (image, not text — read the README claim verbatim).

| Benchmark | Reported result | Verification |
|---|---|---|
| LongMemEval | State-of-the-art "as of January 2026" | Independently reproduced by **Virginia Tech Sanghani Center** and **The Washington Post** per the README |
| LoCoMo, LongMemEval, BEAM, coding-agent suite | Run through AMB (`vectorize-io/agent-memory-benchmark`) from `hindsight-system-evals/` (`CLAUDE.md:90-104`); the old `scripts/benchmarks/run-locomo.sh` is gone | Self-reported, though AMB is a separate repo with its own judge |
| Internal perf | Mock-LLM + pg0 harness (`scripts/benchmarks/run-perf-test.sh`) | Self-reported |

Mark these as **self-reported by the vendor**, with the caveat that Hindsight has the cleanest third-party reproduction story of any system in this study (most others — Mem0 included — are vendor-self-reported only, see [`Profile__StateBench.md`](./Profile__StateBench.md) for why that matters). Independent harness verification at the StateBench level is still the missing rung.

## Coding agents: one package replaced the Claude Code plugin

At the previous pin, Claude Code, Codex, opencode, Cline, Copilot CLI, Cursor CLI, and ZCode each had their own integration package under `hindsight-integrations/`, and the Claude Code one (`hindsight-memory`) was listed in a root `.claude-plugin/marketplace.json`. Those packages are gone. `hindsight-integrations/coding-agents/` (`@vectorize-io/hindsight-coding-agents`, v0.8.8 at this pin, introduced in 0.9.0 on 2026-07-31) replaced them, and commit `81232d062` (2026-10-23) deleted the per-agent sources, their CI and release wiring, and the root Claude Code marketplace. The old npm/PyPI packages were not marked deprecated; they simply stopped being updated.

How it works, from `hindsight-integrations/coding-agents/`:

- **One install command, many harnesses.** `npx @vectorize-io/hindsight-coding-agents install claude-code` (or `all`) wires each agent natively. For Claude Code that is three hooks in `~/.claude/settings.json` (SessionStart, UserPromptSubmit, Stop), an MCP server added at user scope, and a companion skill (`README.md:51-100`). The README lists about 22 harnesses, including Codex CLI, opencode and opencode 2, Cursor CLI, GitHub Copilot CLI, Cline CLI, Devin CLI, Kilo, Qwen Code, Kimi Code, Factory Droid, Antigravity CLI, pi, and DeepSeek Harness. Each harness is a thin entry point (`src/claude-hook.ts`, `src/codex-hook.ts`, ...) over a shared core in `src/core/`.
- **Reflect once, then recall.** The prompt hook recalls on every prompt and runs `reflect` once per session on the first prompt, caching the result (`src/claude-hook.ts:1-13`, `src/core/hook.ts`). The reflect query instructs the bank to report past decisions as declarative, dated, attributed facts and never as instructions, after an incident where reflect output read like a directive (`src/core/inject.ts:1-35`). The injected block is wrapped in a `<hindsight_memory>` tag that transcript readers strip, so the session write-back does not re-ingest it (`src/core/inject.ts:74-80`).
- **Automatic ingestion, no setup command.** Session start spawns a background "deepen" engine that does whatever work is missing: bank config, conversation import, a one-time seed from recent commit messages, then per-commit diff batches, then pages (`README.md:1116-1137`, `src/core/git.ts`). The Stop hook writes the session transcript back to the bank (`src/core/retain-hook.ts`).
- **One bank per repository, shared by every agent.** The default bank id is `coding-agent::{gitProject}`, worktree-aware, so Claude Code, Codex, and opencode working the same repo share one memory (`README.md:1002-1028`). Documents carry `source:` and `harness:` tags for attribution, and the package retains with `observationScopes: "shared"` so consolidation builds one set of beliefs per repo rather than one per agent (`README.md:1054-1078`).
- **Pages.** Agents read curated pages (architecture, conventions, in-flight initiatives) through `hindsight_search_pages`, `hindsight_list_pages`, and `hindsight_read_page` (renamed from `*_knowledge_page*` in 1.2.1).
- **Where memory lives.** Install asks once: Hindsight Cloud (default), a self-hosted server, or a local `hindsight-embed` daemon on `127.0.0.1:9077` shared by all agents (`README.md:501-562`). Daemon settings keep the old Claude Code plugin's names.
- **Migration.** `install` adopts the server URL from an old `~/.hindsight/claude-code.json` or `codex.json`; `--import-conversations` re-imports past Claude Code, Codex, DeepSeek Harness, pi, and Prime Agent sessions from local transcripts, matched to a repo only when the session recorded its working directory (`README.md:447-500`). The old plugin's default was one static `claude_code` bank for everything; the old recall/retain tuning settings are not carried over.
- **Vendor-reported effect.** On AMB's sdebench (61 repository tasks, 3 runs), the README reports accuracy unchanged and corrections per task and cost per task lower with memory, for example Claude Code 0.85 → 0.36 corrections and $0.445 → $0.338 per task (`README.md:24-50`). These are the vendor's numbers.

For this study, the move matters because it turns Hindsight's coding-agent story from "a memory plugin per agent" into "a per-repo memory shared across agents", with git history as a first-class input. That is a closer neighbor to `context-vigilance` (per-repo, git-derived, curated pages) than the old plugin was.

## What's inside this submodule

The repo is a **uv workspace** (`pyproject.toml:2`) plus npm workspaces, about 35 top-level directories. Orientation map:

| Path | What's there |
|---|---|
| `hindsight-api-slim/` | **The server.** FastAPI app, the entire memory engine, Alembic migrations (PG + Oracle). The most important directory. |
| `hindsight-api-slim/hindsight_api/engine/` | Memory engine — `memory_engine.py` (25,057 lines at the current pin, the orchestrator), `memories/` (the pluggable memories store, Postgres default under `memories/pg/`), `retain/`, `search/`, `reflect/`, `consolidation/`, `entity_resolver.py`, `cross_encoder.py`, `embeddings.py`, `llm_wrapper.py` |
| `hindsight-api-slim/hindsight_api/api/` | HTTP layer (`http.py` 10,443 lines) + MCP server (`mcp.py`) |
| `hindsight-api-slim/hindsight_api/alembic/` | Migrations; dialect dispatcher; ~120 versioned files |
| `hindsight-api/` | The "full" API package (slim is the actively developed core; `hindsight-api` wraps it for distribution) |
| `hindsight-all/`, `hindsight-all-slim/`, `hindsight-all-npm/` | Embedded distributions — bundle API + pg0 for `pip install hindsight-all` / `npm install` shapes |
| `hindsight-clients/python/` | Python SDK (`hindsight-client` on PyPI); thin wrapper around generated `hindsight_client_api` |
| `hindsight-clients/typescript/` | TS/Node SDK (`@vectorize-io/hindsight-client` on npm); generated from OpenAPI |
| `hindsight-clients/rust/` | Rust SDK; used by the CLI |
| `hindsight-clients/go/` | Go SDK |
| `hindsight-cli/` | Rust CLI (`cargo`); progenitor-based API client |
| `hindsight-control-plane/` | Next.js admin UI (the `:9999` port) — banks, memories, mental models, recall debugger |
| `hindsight-docs/` | Docusaurus site; the `developer/` directory is the canonical reference |
| `hindsight-embed/` | Embedding service — local sentence-transformers or remote TEI; CLI + daemon |
| `hindsight-dev/` | Benchmarks (`benchmarks/`), upgrade tests, developer tooling |
| `hindsight-system-evals/` | Accuracy benchmarks through AMB |
| `hindsight-system-tests/` | Black-box end-to-end tests against embedded Postgres and a stub provider |
| `hindsight-extensions/` | Example server extensions (`bank-llm-routing`, `static-keys-tenant`, `supabase-tenant`) |
| `hindsight-integrations/` | 49 integrations. Coding agents go through one package, `coding-agents/` (see below); the per-agent `claude-code`, `codex`, `opencode`, `cline`, `copilot-cli`, `cursor-cli`, and `zcode` packages were deleted on 2026-10-23. Others include `litellm`, `crewai`, `langgraph`, `pydantic-ai`, `ag2`, `autogen`, `openai-agents`, `llamaindex`, `claude-agent-sdk`, `google-adk`, `haystack`, `openhands`, `aider`, `zed`, `continue`, `roo-code`, `obsidian`, `zapier`, `n8n`, `dify`, `pipecat`, `agno`, `agentcore`, `paperclip`, `smolagents`, `strands`, `nemoclaw`, `vapi`, `chat`, `ai-sdk` |
| `hindsight-integration-tests/` | End-to-end tests across the integrations |
| `cookbook/` | Worked recipes |
| `helm/hindsight/` | Kubernetes Helm chart |
| `docker/` | Multi-arch images and compose files |
| `monitoring/` | Grafana / Prometheus configs |
| `skills/` | Five agent skills: `hindsight-architect`, `hindsight-cloud`, `hindsight-docs`, `hindsight-local`, `hindsight-self-hosted`, distributed via `npx skills add` |
| `.cursor-plugin/` | Marketplace manifest listing the `grok-bot` plugin. The root `.claude-plugin/` marketplace was deleted on 2026-10-23 when its last plugin moved into `coding-agents` |

If you only read one file: `hindsight-api-slim/hindsight_api/models.py` (351 lines, the core SQLAlchemy schema). If you read two, add `engine/memory_engine.py` for the orchestrator. If you want the storage thesis in prose, read `hindsight-docs/docs/developer/storage.md`.

## Mental model for using it well

- **Pick the bank boundary deliberately.** One bank per "brain" — could be per-user, per-agent-per-user, or per-tenant. Cross-bank queries are the client's job. Disposition belongs to the bank, so if two users need different agent personalities you need two banks.
- **Trust the taxonomy.** Don't force everything into `world`. First-person agent actions ("I ran the migration", "I noticed the user prefers terse responses") are `experience` (or `assistant` at the LLM layer) — keeping the split makes recall ranking and reflect's hierarchical tool surface work correctly.
- **Use `reflect` for anything synthesis-shaped, `recall` for anything fact-shaped.** Recall is a 4-way parallel retrieval with RRF + reranking — fast, deterministic. Reflect is an agentic LLM loop — slower, expensive, but it's where mental models and observations earn their keep.
- **Watch the consolidation toggle per bank.** It's the eviction-substitute and the supersession-substitute. If you disable it (`enable_observations=False`, `consolidator.py:1904`), you lose both — recall will surface contradictory facts side-by-side.
- **Don't fight the "one Postgres" decision.** It's the system's biggest opinion. If you need a real graph DB or a separate vector store, you're using the wrong tool.
- **Curate mental models for high-traffic queries.** Auto-consolidation produces observations; mental models are user-curated and sit above observations in the reflect hierarchy. Pinning a mental model is how you tell the agent "here's the authoritative answer for this kind of question."

## When NOT to reach for this

- **You want a tiny in-process memory layer.** Hindsight is a full stack — even `hindsight-all` embedded mode boots pg0 inside your process. For a 100-line agent, Mem0's library mode or your own SQLite is simpler.
- **You need explicit supersession chains** (compliance, agent-medical, agent-legal where you must point at "this fact superseded that one"). Hindsight's consolidation rewrites observations but does not carry a pointer chain on the underlying facts. Use [[Profile__Neo]].
- **You're allergic to LLM-driven extraction.** Retain calls an LLM per chunk; consolidation calls another. For high-throughput pipelines this is a real cost line.
- **Your memory is "agent's filesystem of notes" not "structured memory of a user".** The bank+taxonomy model assumes there's a coherent subject (a user, an agent) the memories belong to. For freeform notes, [[Profile__MemPalace]] is closer.
- **You need a graph DB you can run separate Cypher queries against.** The graph here lives in Postgres CTEs walking `memory_links`. Powerful enough for recall, not a substitute for Neo4j/Memgraph if graph IS the product.
- **Local markdown notes are enough.** Same caveat as the Mem0 profile — for human-readable durable project knowledge, the `context-vigilance` skill (markdown + frontmatter + git) is dramatically simpler and survives without any running service.

## How this compares to the rest of the study

| Axis | Hindsight | Mem0 | Neo | MemPalace |
|---|---|---|---|---|
| **Shape** | Full stack (server + 4 SDKs + CLI + UI + Helm) | Library + server + hosted | Code-reasoning memory + agent | Filesystem-shaped notes |
| **Write axis** | **Typed taxonomy** (world / experience / observation) enforced at DB CHECK | Append + NER entity link | Append + supersession chain | Append (no extraction) |
| **Storage** | Single Postgres (pgvector + tsvector + JSONB + CTE graph) | Vector + entity store + SQLite | Scoped JSON files | Markdown + filesystem |
| **Retrieval** | 4-way parallel (semantic / BM25 / graph / temporal) + RRF + cross-encoder + multiplicative boosts | Hybrid (semantic + BM25 + entity-link) | Semantic + confidence × success | Grep + structure walk |
| **Supersession** | Implicit via consolidation rewriting observations; manual invalidation via curation | None in OSS | First-class `superseded_by` | None |
| **Eviction** | Consolidation deletes orphan observations; raw memories never auto-deleted, but can be invalidated into an archive | None in OSS | Stale-prune + demotion | None |
| **Scope model** | **Bank** (with disposition + mission) | user/agent/run/actor | global/org/project/session | Directory tree |
| **Distinguishing verb** | **`reflect`** — agentic loop over own memories with native tool calls | `search` (CRUD-shaped) | Single recall surface | File ops |
| **Self-reported SOTA** | LongMemEval (independently reproduced) | LoCoMo, LongMemEval (vendor) | n/a | n/a |
| **License** | MIT | Apache-2.0 | (see Profile) | (see Profile) |

## How this compares to Mem0 specifically

Both ship the same hybrid-retrieval thesis (vector + BM25 + entity/graph), both expose a small verb surface, both treat scope as first-class. The genuine architectural divergence is **the write axis**:

- **Mem0** treats memory as one kind of thing (`memory_type` is optional metadata) and bets that *retrieval ranking* sorts truth from noise.
- **Hindsight** treats memory as having a *fundamental type* (epistemic vs. experiential vs. inferential) and bets that *the write should be classified up front* — because the right retrieval strategy, the right index, even the right LLM extraction prompt all differ per type.

The reflect-vs-search divergence follows from that. Mem0's `search` returns memories; the agent decides what to do with them. Hindsight's `reflect` lets the bank *think about itself* — synthesize across observations, walk to mental models, run an agentic tool loop — because once memory has structure, structured introspection becomes possible.

Both can be right. If "this user mentioned they like coffee" is undifferentiated from "I refactored the API yesterday" in your domain, Mem0's flatter model is cheaper. If those are different categories of thing your agent should reason about differently, Hindsight's typed model earns its complexity.

## How this compares to our own `context-vigilance` skill

Same caveat as the Mem0 profile. The `context-vigilance` skill is a human-readable markdown discipline — directory roles, frontmatter, wikilinks, versioning — meant for human + agent collaborative knowledge. It assumes the artifacts are reviewed in PRs and live in git.

Hindsight is the right tool when the consumer is an agent, the corpus is large enough that a human can't curate it, and the value of typed extraction + 4-way retrieval + `reflect`-style introspection outweighs the cost of running a Postgres-backed service. They coexist cleanly: `context-vigilance` for durable project knowledge a human reads, Hindsight for the per-user/per-agent memory the system accumulates while operating.

## Changes since the previous pin (2026-05-26 → 2026-11-21)

- Old pin `30acca6fd9d8`, new pin `62a03bfc96f1`, 2,768 commits. Core releases in range: v0.7.0 through v0.10.3, then **v1.0.0 (2026-10-13)**, v1.1.0, v1.2.0, v1.2.1, v1.2.2, v1.2.3 (2026-11-19). Core changelog: `hindsight-docs/src/pages/changelog/index.md`; each integration has its own under `hindsight-docs/src/pages/changelog/integrations/`. License unchanged (MIT, `LICENSE`).
- **Coding Agents replaced the per-agent plugins**, including Claude Code. See the section above (`hindsight-integrations/coding-agents/`).
- **1.0.0** itself is small: tag-scoped API callers are rejected from bank-wide operations, a 503 with Retry-After when the store is unavailable, and provider tracing. The larger changes landed in the 0.8 to 0.10 line and in 1.1 and 1.2.
- **Decision models (1.1.0 experimental, 1.2.0 final names).** A small typed-answer model, `hs-decide-1-flash`, can settle consolidation, recall ranking and deduplication, reflect orchestration, and mental-model refresh decisions before an LLM call (`README.md:364-381`). Vendor-published weights on Hugging Face.
- **Reflect** now makes tool calls as replies to a response schema instead of provider-native tool calling (1.2.0 breaking change, `engine/reflect/agent.py:2`), adds `read_mental_models` and `answer_ready`, and enforces "recall before saying nothing is known" through `answer_found` (`engine/reflect/tools_schema.py:198-214`).
- **Recall defaults changed.** Omitting `types` now recalls observations too (`api/http.py:5578, 5617`). New options: `prefer_observations`, configurable recency decay, per-bank switches for graph, temporal, reranking, and text search, live pruning and deduplication.
- **Extraction boundary moved.** User preferences, rules, and corrections are now `world` facts even when stated to the assistant (`engine/retain/fact_extraction.py:1115-1117`).
- **Curation (0.8.2).** Facts can be edited, invalidated into `invalidated_memory_units`, and restored (`hindsight-docs/docs/developer/api/memories.mdx:101-125`). This softens the profile's earlier "no soft-delete" description.
- **Pluggable memories store (2026-07-30).** `engine/memories/base.py` puts the memory tables behind `MemoriesExtension`; Postgres stays the default and sole tested implementation.
- **Pages.** "Knowledge Pages" (0.9.0) became "Pages" in 1.2.1; they are mental models presented as a wiki-like tree, projectable to markdown, with an Observation Tree export as an Open Knowledge Format bundle of Markdown files (1.2.2).
- **Experimental Knowledge Banks** (1.2.2): document and multimodal passage search with their own per-bank vector index, behind a feature flag.
- **Other:** inline image and file attachments on retain (0.10.0), bank aliases, clone, rename, export/import, observation scopes, Memory Defense secret/PII scanning, webhooks with signatures, many more LLM providers (including subscription-backed Codex, Claude Code, Cursor, and Copilot), and benchmarks moved to AMB under `hindsight-system-evals/`.
- Line citations throughout this profile were re-located at the new pin. `engine/search/link_expansion_retrieval.py`, `hindsight-tools/`, `.claude-plugin/`, and `scripts/benchmarks/run-locomo.sh` no longer exist, and the bank `retain_mission` / `retain_extraction_mode` fields are no longer in `engine/response_models.py`; the profile now cites their replacements.

## One-line summary

> Hindsight is a typed-taxonomy agent memory system (world / experience / observation / mental-model) with a three-verb API where `reflect` runs an agentic loop over the agent's own memories, 4-way parallel retrieval (semantic + BM25 + graph + temporal) fused by RRF and cross-encoder reranking on a single Postgres instance, shipped as a complete open-source stack — and the only system in this study with claimed third-party benchmark reproduction.
