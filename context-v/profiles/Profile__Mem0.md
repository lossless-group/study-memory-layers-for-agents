---
name: Mem0 Profile
slug: mem0
upstream: https://github.com/mem0ai/mem0
package: mem0ai (PyPI), mem0ai (npm)
license: Apache-2.0
maintainer: Mem0.ai (Taranjeet Singh et al.)
study: studies/memory-layers-for-agents
profile_path: studies/memory-layers-for-agents/mem0
profile_kind: library + server + hosted-platform
date_created: 2026-05-17
---

# Mem0 — Profile

A profile of Mem0 as it lives in this study (`studies/memory-layers-for-agents/mem0/`). Cites pinned paths so you can jump to source rather than trust paraphrase. Read this alongside [`Profile__Neo.md`](./Profile__Neo.md), [`Profile__Volt.md`](./Profile__Volt.md), and [`Profile__StateBench.md`](./Profile__StateBench.md) — the four define almost the whole design space the study cares about.

## TL;DR

Mem0 is the most-starred (54k+) **general-purpose memory layer** for AI agents. The thesis: agents need a *layer*, not a *framework* — a small surface (`add`, `search`, `get`, `update`, `delete`) that any agent in any stack can call, with the messy details (extraction, embedding, ranking, dedup, scope) hidden behind it.

It is a **hybrid store** under the hood (`mem0/memory/main.py:340-411`): a primary vector store (Qdrant by default, ~30 providers supported) holds the memories themselves; a secondary **entity store** (same vector backend, separate collection) holds NER-extracted entities (PER/ORG/LOC) with `linked_memory_ids` arrays that wire each entity to every memory that mentions it; a local **SQLite `history.db`** (`mem0/memory/storage.py:102-149`) holds the audit trail (`history` table) and the conversation transcript (`messages` table). There is **no graph database** in the OSS path despite the "graph memory" marketing — the graph is implicit in those entity arrays. The hosted platform adds an actual Neo4j tier.

The 2026 v3 write algorithm (`mem0/memory/main.py:699-850`, `mem0/configs/prompts.py:468`) is **append-only with entity linking** and a **single LLM call per turn** — no separate UPDATE/DELETE phase, no agentic loops. That single-pass design is what unlocks the headline benchmark numbers (`README.md:45-68`): 91.6 on LoCoMo, 93.4 on LongMemEval, p50 latency around 1 second.

If you want one sentence: **Mem0 is a vector store with NER-linked entities, a tight CRUD-shaped API, three deployment modes (library / self-hosted server / hosted), and a benchmark-driven write algorithm tuned to do as little LLM work per turn as possible.**

## Why this exists — the design bet

Most "agent memory" prior to Mem0 was either (a) a buffer the framework managed for you (LangChain `ConversationBufferMemory` etc.) or (b) you-roll-your-own on top of a vector DB. Mem0's bet is that there's a stable, opinionated layer in between worth shipping as a product:

1. **Scope and namespacing are first-class.** Every call requires at least one of `user_id`, `agent_id`, `run_id` (`mem0/memory/main.py:231-314`). The library refuses to let you forget which user this memory belongs to.
2. **Extraction is implicit and configurable.** You hand it a `messages` array (chat-shaped) and it figures out what's worth remembering. The `infer=False` escape hatch lets you bypass that and write a literal memory if you already know.
3. **Recall is hybrid, not pure-vector.** Three signals (semantic + BM25 + entity-link boost) fused additively (`mem0/utils/scoring.py:60-121`). Vector-only retrieval is the most common mistake in this space, and Mem0 explicitly refuses to make it.
4. **Deployment is decoupled from the API.** Same surface whether you `pip install mem0ai` (in-process), run the Docker server, or hit `api.mem0.ai`.

## Storage topology

| Layer | Backend | Purpose | Code |
|---|---|---|---|
| Primary | Vector store (Qdrant default; 30 providers) | Memory text + embedding + scope metadata | `mem0/memory/main.py:340-342` |
| Entity store | Same vector backend, second collection | NER entities with `linked_memory_ids` arrays | `mem0/memory/main.py:390-411` |
| History | Local SQLite `~/.mem0/history.db` | Audit log + raw message transcript | `mem0/memory/storage.py:102-149` |
| Graph (hosted only) | Neo4j | Cross-memory graph; not in OSS library | `server/` Docker compose |

The "vector + entity + audit" triad is the load-bearing design. Vector handles fuzzy recall, the entity store handles "what do you know about Alice", and SQLite is the only thing a non-AI program can fully parse without provider-specific tooling.

## Schema of a memory record

The vector-store payload (`mem0/memory/main.py:1593-1599`):

```python
{
    "id": "uuid",
    "data": "memory text",
    "hash": "md5 hex digest",          # dedup key
    "created_at": "ISO 8601 UTC",
    "updated_at": "ISO 8601 UTC",
    "text_lemmatized": "...",          # for BM25
    "user_id": "...", "agent_id": "...", "run_id": "...",
    "actor_id": "alice",               # optional
    "role": "user|assistant",          # optional
    "memory_type": "procedural_memory" # optional
}
```

The entity-store payload (`mem0/memory/main.py:442-446`):

```python
{
    "data": "Alice",
    "entity_type": "PERSON",           # spaCy NER label
    "linked_memory_ids": ["uuid1", "uuid2", ...],
    "user_id": "...", "agent_id": "...", "run_id": "..."
}
```

The API response shape (`mem0/configs/base.py:16-26`) flattens this into a `MemoryItem` with `id`, `memory`, `hash`, `metadata`, `score`, `created_at`, `updated_at`.

Notable for what's **missing**: no confidence field, no supersession pointers, no "is_valid" flag, no graph edge table. Compare to Neo (`Profile__Neo.md` §schema) which carries all of those — Mem0's record is intentionally flatter.

## Write policy — the v3 single-pass algorithm

Pre-v3 Mem0 had a two-phase ADD-then-UPDATE loop that asked the LLM "should we update or merge existing memories?" v3 (April 2026, `README.md:45-68`) replaced that with one prompt — `ADDITIVE_EXTRACTION_PROMPT` (`mem0/configs/prompts.py:468`) — that extracts all memorable facts from a turn in a single call. Hash-based dedup (`mem0/memory/main.py:784-790`) silently skips memories already present. Manual `update(memory_id, data)` (`mem0/memory/main.py:1657-1685`) is the only in-place mutation path; the LLM never overwrites memories automatically.

The trade-off: you get LoCoMo 91.6 and p50 ~1s, but the system has **no automatic supersession** in the OSS path. If the user says "actually my birthday is March 4, not May 4" you get two memories, both true-looking. Mem0's bet is that **recall ranking** handles this — newer/higher-scored memory wins the surface — and that explicit `update`/`delete` calls fill the gap when it doesn't.

## Recall API

```python
memory.search(
    query="what do you know about my preferences?",
    top_k=20,
    filters={"user_id": "alice"},       # at least one of user/agent/run required
    threshold=0.1,                       # semantic-similarity gate
    rerank=False,                        # optional Cohere/HF/LLM reranker
)
```

Three signals fused additively (`mem0/utils/scoring.py:60-121`):

```
combined_score = (semantic + bm25 + entity_boost) / max_possible
max_possible   = 1.0 + (1.0 if bm25 else 0) + (0.5 if entity_boost else 0)
```

Over-fetch is `4 * top_k` from the vector store (`mem0/memory/main.py:1343-1425`); BM25 and entity boost are applied on that candidate set. Filter operators include `eq/ne/gt/in/contains` plus boolean `AND/OR/NOT` (`mem0/memory/main.py:1142-1197`) — richer than the typical "metadata filter" most vector DBs ship with.

Other methods (`add`, `get`, `get_all`, `update`, `delete`, `delete_all`, `history`) round out a deliberately CRUD-shaped surface.

## Eviction & compaction

**There is none in the OSS library.** No TTL, no summarizer, no soft-delete reconciliation. Memories accumulate; the only built-in cleanup is `make prune-logs` on the self-hosted server's request log (`server/README.md:77-86`). OpenMemory (the alt stack under `openmemory/`) defines `active|paused|archived|deleted` states (`openmemory/api/app/models.py:30-34`) and an `archive_policies` table with `days_to_archive`, but **the auto-trigger is not implemented in the code I read** — it's schema-only.

This is a real gap if you're operating a large-scale memory layer. Compare to Volt's deterministic three-level escalation (`Profile__Volt.md` §thresholds) or Neo's `prune_stale_facts` / `demote_unhelpful_facts` (`Profile__Neo.md` §eviction).

## Scopes & namespacing

Three required scopes (at least one): `user_id`, `agent_id`, `run_id` (`mem0/memory/main.py:231-314`). Optional `actor_id` lets you tag who said what inside a multi-party session. Session-scope string in SQLite is the deterministic `"user_id=u1&agent_id=a1&run_id=r1"`.

This is the API's most opinionated choice. Most vector-DB-as-memory hacks treat scope as "just another metadata field." Mem0 treats it as the central indexing axis and refuses calls that don't specify one — which sounds annoying but eliminates the entire class of "I accidentally returned Alice's memories to Bob" bugs.

## Operational story

Three deployment modes, same API:

1. **Library** (`pip install mem0ai`). Pure in-process; SQLite at `~/.mem0/history.db`; vector store can be in-process (FAISS, local Qdrant) or remote (Pinecone, pgvector). The agent is stateless; memory is external.
2. **Self-hosted server** (`server/`). `docker-compose up` brings FastAPI (8888) + Postgres (8432) + Neo4j (8687). Dashboard at :3000, JWT login, `X-API-Key` for clients. `make bootstrap` creates admin user + password + key.
3. **Hosted platform** (`api.mem0.ai`). `MemoryClient(api_key=...)`. Multi-tenant (org_id + project_id derived from key); graph memory (Neo4j), webhooks, audit logs, managed ops are platform-only features.

Telemetry is on by default (`MEM0_TELEMETRY=true`) — note this for any privacy-sensitive deployment.

## Headline benchmark numbers

Source: `README.md:45-68` (April 2026 v3).

| Benchmark | v2 → v3 | Tokens | p50 |
|---|---|---|---|
| LoCoMo | 71.4 → **91.6** | 7.0K | 0.88s |
| LongMemEval | 67.8 → **93.4** | 6.8K | 1.09s |
| BEAM (1M) | — → **64.1** | 6.7K | 1.00s |
| BEAM (10M) | — → **48.6** | 6.9K | 1.05s |

Methodology lives under `evaluation/` (`evaluation/README.md:19-22, 150-172`) and compares against [LoCoMo](https://github.com/snap-research/locomo) paper baselines, ReadAgent, MemoryBank, MemGPT, A-Mem, LangMem, vanilla RAG, full-context, OpenAI's built-in memory, and Zep. The harness is open enough that StateBench-style independent verification is feasible — see [`Profile__StateBench.md`](./Profile__StateBench.md) for why you should want that.

## What's inside this submodule

| Path | What's there |
|---|---|
| `mem0/` | Core Python SDK (`mem0ai`) — memory class, store/LLM/embedder/reranker abstractions |
| `mem0-ts/` | TypeScript SDK |
| `cli/{python,node}/` | CLI tools |
| `server/` | Self-hosted FastAPI + Postgres + Neo4j stack |
| `openmemory/` | Alternative self-hosted memory platform (FastAPI + Qdrant + Next.js UI) |
| `evaluation/` | LoCoMo / LongMemEval / BEAM benchmark code |
| `mem0-plugin/`, `openclaw/` | Editor plugins (Claude Code, Cursor, Codex) via MCP |
| `vercel-ai-sdk/` | Vercel AI SDK integration |
| `examples/`, `cookbooks/` | Sample projects and notebooks |
| `docs/` | Mintlify docs site |

If you only read one file: `mem0/memory/main.py`. The whole behavioral surface is there — extraction, storage, search, scoring — in roughly 2,000 lines.

## Mental model for using it well

- **Always pass a scope.** Even for a prototype. The day you bolt on multi-user is the day "I forgot to filter" becomes an incident.
- **Use `infer=False` when you already know.** "User confirmed email is alice@example.com" doesn't need an extraction LLM call — write it literally.
- **Don't expect supersession.** If correctness over time matters (a "what's my current address" workflow), build explicit `update` paths or read [`Profile__StateBench.md`](./Profile__StateBench.md) to understand the failure modes Mem0 is structurally exposed to.
- **Treat the entity store as the killer feature.** Pure vector recall plateaus quickly. The entity-link boost is what makes "tell me what we know about X" work without a real graph DB.
- **Use the self-hosted server before reaching for the platform.** The OSS dashboard is good enough for most teams; the hosted tier earns its keep when you need Neo4j-backed graph features or audit-grade ops.

## When NOT to reach for this

- **You need provenance and supersession** (compliance, agent-medical, agent-legal). Mem0's append-only-with-no-supersession will resurrect superseded facts. Use a state-based system (see Memgine in `Profile__StateBench.md`) or Neo (with its `superseded_by` chain).
- **You're a coding agent and want full immutable history.** That's Volt's job. Mem0 forgets what it didn't extract.
- **You want a graph-first model.** Use Graphiti/Zep (candidate in study README) — entity-link arrays are not a graph.
- **You're allergic to LLM-driven extraction.** Mem0 calls an LLM on every `add()`. For high-throughput pipelines this is a real cost line.
- **Local markdown notes are enough.** If your "memory" needs are "I want to look stuff up in my own notes," our own `context-vigilance` skill (markdown + wikilinks + git) is dramatically simpler and produces artifacts a human can read. Mem0 only earns its keep once you're routing agent calls.

## How this compares to the rest of the study

| Axis | Mem0 | Neo | Volt | StateBench (as a yardstick) |
|---|---|---|---|---|
| **Shape** | General memory layer | Code-reasoning memory + agent | Coding agent with built-in memory | Conformance benchmark |
| **Storage** | Vector + entity store + SQLite | Scoped JSON files w/ inline embeddings | Postgres DAG (immutable log + summary nodes) | N/A |
| **Write policy** | Append-only + entity link | Append-only + deterministic supersession (cosine > 0.85) | Verbatim append + async summary | Tests this directly |
| **Eviction** | None in OSS | Stale-prune + demotion + synthesis | Dolt evicts bindles; Upward never evicts | Penalizes wrong eviction |
| **Supersession** | Not modeled | First-class | First-class via immutable log | Measures resurrection rate |
| **Scope model** | user/agent/run/actor (required) | global/org/project/session (auto-detected) | conversation (per session) | Tests scope leak |
| **Recall** | Hybrid (semantic + BM25 + entity) | Semantic + confidence × success | Bindle expansion + `lcm_grep` | Tests must/must-not mention |
| **Operational story** | Library / server / hosted | Local CLI + MCP plugin | Local terminal + embedded Postgres | Harness / Hub Space |
| **Best fit** | Any agent that needs cross-session user memory | Code assistants that improve over time | Long-horizon coding sessions | Evaluating any of the above |

## How this compares to our own `context-vigilance` skill

Worth being explicit because the temptation to over-reach is real. The `context-vigilance` agent-skill is a *human-readable markdown discipline* — directory roles (`prompts/`, `specs/`, `blueprints/`, `reminders/`, `explorations/`, `issues/`), frontmatter, epoch.major.minor.patch versioning, wikilinks. It is **not** a memory layer; there is no embedding, no recall API, no scope-aware filter, no extraction. It assumes a human is in the loop and the artifacts are read by other humans (and by agents that grep + read markdown).

Mem0 is the right tool when: the consumer is an agent, the corpus is too large to keep in context, the scope axis is `user_id`/`agent_id` (not "this project's docs"), and the value of fuzzy recall outweighs the cost of LLM extraction. `context-vigilance` is the right tool when: the consumer is a human + an agent that can read the repo, the artifacts need to be reviewed in PRs, and the structure matters as much as the content.

They can coexist: `context-vigilance` for the durable project knowledge, Mem0 for ephemeral per-user/per-session memory the agent accumulates while operating.

## One-line summary

> Mem0 is a hybrid vector + entity store with a CRUD-shaped agent-memory API, a single-pass write algorithm tuned for benchmark wins, three deployment modes that share one surface, and a deliberate refusal to model supersession — which makes it the right default for general agent memory and the wrong default for anything where state-correctness-over-time is the failure mode.
