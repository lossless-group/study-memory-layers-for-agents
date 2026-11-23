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
site_uuid: 23d5b3f3-c3f2-4390-8a81-a48eb6f100ca
hex_code: qt8j06
date_authored_initial_draft: 2026-05-17
date_authored_current_draft: 2026-11-23
lede: >-
  Despite the graph-memory marketing, the graph is an array of linked memory
  IDs hanging off spaCy-extracted entities.
summary: >-
  Profile of Mem0 (mem0ai) as pinned in the memory-layers-for-agents study. It
  is the field's default general-purpose memory layer and the baseline every
  other entry positions against. Covers the vector + entity-store + SQLite
  triad, the memory and entity payload schemas, the v3 single-pass additive
  extraction that replaced the older ADD-then-UPDATE loop, additive hybrid
  scoring across semantic + BM25 + entity boost, the mandatory
  user/agent/run scope axis, the absence of any eviction policy in OSS, and the
  three deployment modes. Read the write-policy and eviction sections together —
  they are where the benchmark wins and the structural gaps come from the same
  design choice, and they set up Profile__StateBench, Profile__Neo, and
  Profile__RetainDB as the counterarguments.
publish: true
title: Mem0 — Profile
date_modified: 2026-11-23
authors:
  - Michael Staton
status: Reference
type: Profiles
---

# Mem0 — Profile

A profile of Mem0 as it lives in this study (`studies/memory-layers-for-agents/mem0/`). Cites pinned paths so you can jump to source rather than trust paraphrase. Read this alongside [`Profile__Neo.md`](./Profile__Neo.md), [`Profile__Volt.md`](./Profile__Volt.md), and [`Profile__StateBench.md`](./Profile__StateBench.md) — the four define almost the whole design space the study cares about.

## TL;DR

Mem0 is the most-starred (54k+ at the original May 2026 pin; not re-verified) **general-purpose memory layer** for AI agents. The thesis: agents need a *layer*, not a *framework* — a small surface (`add`, `search`, `get`, `update`, `delete`) that any agent in any stack can call, with the messy details (extraction, embedding, ranking, dedup, scope) hidden behind it.

It is a **hybrid store** under the hood (`mem0/memory/main.py:487-590`): a primary vector store (Qdrant by default, ~30 providers supported) holds the memories themselves; a secondary **entity store** (same vector backend, separate collection) holds NER-extracted entities (PER/ORG/LOC) with `linked_memory_ids` arrays that wire each entity to every memory that mentions it; a local **SQLite `history.db`** (`mem0/memory/storage.py:102-148`) holds the audit trail (`history` table) and the conversation transcript (`messages` table). There is **no graph database** in the OSS path despite the "graph memory" marketing; the graph is implicit in those entity arrays. Since August 2026 the docs say so directly: graph memory is framed as a Platform feature, removed from OSS (PR #6808).

The 2026 v3 write algorithm (`mem0/memory/main.py:760-1050`, `mem0/configs/prompts.py:468`) is **append-only with entity linking** and a **single LLM call per turn** — no separate UPDATE/DELETE phase, no agentic loops. That single-pass design is what Mem0 credits for its headline benchmark numbers (`README.md:56-80`, vendor-reported): 92.5 on LoCoMo, 94.4 on LongMemEval, p50 latency around 1 second. The README now adds that these scores reflect the managed platform, "which includes proprietary optimizations not available in the open-source SDK" (`README.md:65`).

If you want one sentence: **Mem0 is a vector store with NER-linked entities, a tight CRUD-shaped API, three deployment modes (library / self-hosted server / hosted), and a benchmark-driven write algorithm tuned to do as little LLM work per turn as possible.**

## Why this exists — the design bet

Most "agent memory" prior to Mem0 was either (a) a buffer the framework managed for you (LangChain `ConversationBufferMemory` etc.) or (b) you-roll-your-own on top of a vector DB. Mem0's bet is that there's a stable, opinionated layer in between worth shipping as a product:

1. **Scope and namespacing are first-class.** Every call requires at least one of `user_id`, `agent_id`, `run_id` (`mem0/memory/main.py:314-406`, error raised at line 393). The library refuses to let you forget which user this memory belongs to.
2. **Extraction is implicit and configurable.** You hand it a `messages` array (chat-shaped) and it figures out what's worth remembering. The `infer=False` escape hatch lets you bypass that and write a literal memory if you already know.
3. **Recall is hybrid, not pure-vector.** Three signals (semantic + BM25 + entity-link boost) fused additively (`mem0/utils/scoring.py:60-130`). Vector-only retrieval is the most common mistake in this space, and Mem0 explicitly refuses to make it.
4. **Deployment is decoupled from the API.** Same surface whether you `pip install mem0ai` (in-process), run the Docker server, or hit `api.mem0.ai`.

## Storage topology

| Layer | Backend | Purpose | Code |
|---|---|---|---|
| Primary | Vector store (Qdrant default; ~30 providers, Oracle AI Vector Search added) | Memory text + embedding + scope metadata | `mem0/memory/main.py:487-512` |
| Entity store | Same vector backend, second collection (`<collection>_entities`) | NER entities with `linked_memory_ids` arrays | `mem0/memory/main.py:422-424, 560-580` |
| History | Local SQLite `~/.mem0/history.db` | Audit log + raw message transcript | `mem0/memory/storage.py:102-148` |
| Graph (Platform only) | Hosted graph tier | Cross-memory graph; not in the OSS library or the self-hosted server | Platform docs |

The "vector + entity + audit" triad is the load-bearing design. Vector handles fuzzy recall, the entity store handles "what do you know about Alice", and SQLite is the only thing a non-AI program can fully parse without provider-specific tooling.

## Schema of a memory record

The vector-store payload (`mem0/memory/main.py:1022-1040`, `1975-1990`):

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
    "memory_type": "procedural_memory", # optional
    "attributed_to": "...",           # optional, new since May 2026
    "expiration_date": "YYYY-MM-DD"    # optional, new since May 2026
}
```

The entity-store payload (`mem0/memory/main.py:636-642`):

```python
{
    "data": "Alice",
    "entity_type": "PERSON",           # spaCy NER label
    "linked_memory_ids": ["uuid1", "uuid2", ...],
    "user_id": "...", "agent_id": "...", "run_id": "..."
}
```

The API response shape (`mem0/configs/base.py:16-26`) flattens this into a `MemoryItem` with `id`, `memory`, `hash`, `metadata`, `score`, `created_at`, `updated_at`.

Notable for what's **missing**: no confidence field, no supersession pointers, no "is_valid" flag, no graph edge table. The one validity signal added since May 2026 is an optional `expiration_date`. Compare to Neo (`Profile__Neo.md` §schema) which carries all of those — Mem0's record is intentionally flatter.

## Write policy — the v3 single-pass algorithm

Pre-v3 Mem0 had a two-phase ADD-then-UPDATE loop that asked the LLM "should we update or merge existing memories?" v3 (April 2026, `README.md:56-80`) replaced that with one prompt — `ADDITIVE_EXTRACTION_PROMPT` (`mem0/configs/prompts.py:468`) — that extracts all memorable facts from a turn in a single call. Hash-based dedup (`mem0/memory/main.py:1022-1026`) silently skips memories already present. Manual `update(memory_id, text=...)` (`mem0/memory/main.py:1829-1882`; the `data` argument is deprecated in favor of `text`, PR #6044) is the only in-place mutation path; the LLM never overwrites memories automatically. Identity fields are immutable after creation: `update()` metadata can no longer overwrite `user_id`/`agent_id`/`run_id` (PR #6278), and `add()` metadata can no longer set them (PR #6656). An update that races a delete no longer re-creates the deleted memory (PR #7787).

The trade-off: you get LoCoMo 91.6 and p50 ~1s, but the system has **no automatic supersession** in the OSS path. If the user says "actually my birthday is March 4, not May 4" you get two memories, both true-looking. Mem0's bet is that **recall ranking** handles this — newer/higher-scored memory wins the surface — and that explicit `update`/`delete` calls fill the gap when it doesn't.

## Recall API

```python
memory.search(
    query="what do you know about my preferences?",
    top_k=20,
    filters={"user_id": "alice"},       # at least one of user/agent/run required
    threshold=0.1,                       # semantic-similarity gate
    rerank=False,                        # optional Cohere/HF/LLM reranker
    explain=False,                       # new: per-result score_details
    show_expired=False,                  # new: include expired memories
)
```

Three signals fused additively (`mem0/utils/scoring.py:60-130`, `ENTITY_BOOST_WEIGHT = 0.5` at line 57):

```
combined_score = (semantic + bm25 + entity_boost) / max_possible
max_possible   = 1.0 + (1.0 if bm25 else 0) + (0.5 if entity_boost else 0)
```

Over-fetch is `max(4 * top_k, 60)` from the vector store (`mem0/memory/main.py:1655`); BM25 and entity boost are applied on that candidate set, and entity-boost lookups now run in parallel (PR #5377). The combined score is capped at 1.0. Hybrid search logs a warning when it silently degrades to semantic-only (PR #5444). Filter operators include `eq/ne/gt/gte/lt/lte/in/nin/contains/icontains` and `*` wildcard plus boolean `AND/OR/NOT` (`mem0/memory/main.py:1393-1440`, `1538`), richer than the typical "metadata filter" most vector DBs ship with. The `reference_date` (search) and `timestamp` (add) arguments exist but are documented as Platform-only and unsupported in OSS (`mem0/memory/main.py:787, 1435`).

Other methods (`add`, `get`, `get_all`, `update`, `delete`, `delete_all`, `history`) round out a deliberately CRUD-shaped surface.

## Eviction & compaction

**Almost none in the OSS library.** No summarizer, no soft-delete reconciliation, no automatic decay. The one addition since May 2026 is an optional per-memory `expiration_date` (YYYY-MM-DD) on `add()` and `update()` (`mem0/memory/main.py:427-450, 769, 1834`). Expired memories are not deleted; `search()` and `get_all()` hide them unless `show_expired=True` (`mem0/memory/main.py:1369, 1684`). Memories otherwise accumulate. The only built-in storage cleanup is `make prune-logs` on the self-hosted server's request log (`server/README.md:97-98`). OpenMemory, the alternative stack whose `archive_policies` table this profile once noted as schema-only, was removed from the monorepo in July 2026 (PR #6530).

This is a real gap if you're operating a large-scale memory layer. Compare to Volt's deterministic three-level escalation (`Profile__Volt.md` §thresholds) or Neo's `prune_stale_facts` / `demote_unhelpful_facts` (`Profile__Neo.md` §eviction).

## Scopes & namespacing

Three required scopes (at least one): `user_id`, `agent_id`, `run_id` (`mem0/memory/main.py:231-314`). Optional `actor_id` lets you tag who said what inside a multi-party session. Session-scope string in SQLite is the deterministic `"user_id=u1&agent_id=a1&run_id=r1"`.

This is the API's most opinionated choice. Most vector-DB-as-memory hacks treat scope as "just another metadata field." Mem0 treats it as the central indexing axis and refuses calls that don't specify one — which sounds annoying but eliminates the entire class of "I accidentally returned Alice's memories to Bob" bugs.

## Operational story

Three deployment modes, same API:

1. **Library** (`pip install mem0ai`). Pure in-process; SQLite at `~/.mem0/history.db`; vector store can be in-process (FAISS, local Qdrant) or remote (Pinecone, pgvector). The agent is stateless; memory is external.
2. **Self-hosted server** (`server/`). `docker-compose up` brings FastAPI (8888) + Postgres with pgvector (8432) + a dashboard on :3000 (`server/docker-compose.yaml`). JWT login, `X-API-Key` for clients, Alembic migrations, and a rate limiter. There is no Neo4j service in the compose file.
3. **Hosted platform** (`api.mem0.ai`). `MemoryClient(api_key=...)`. Multi-tenant (org_id + project_id derived from key); graph memory, webhooks, audit logs, User Profiles (PR #7340), "Dream" memory synthesis endpoints, BYOK, and managed ops are platform-only. New since May 2026: an agent can mint its own API key with `mem0 init --agent` and a human can claim the account later (`README.md:100-118`). The OSS SDK now prints OSS-to-Platform notices (PR #5494), and a `mem0-oss-to-platform` skill automates migration.

Telemetry is on by default (`MEM0_TELEMETRY` defaults to `"True"`, `mem0/memory/telemetry.py:14`); note this for any privacy-sensitive deployment.

## Headline benchmark numbers

Source: `README.md:56-80`. Vendor-reported. The LoCoMo and LongMemEval figures were revised upward after the previous pin (91.6 → 92.5 and 93.4 → 94.4), at a stated top_200 retrieval budget, and the README now says the scores "reflect Mem0's managed platform, which includes proprietary optimizations not available in the open-source SDK" (`README.md:65`). The README also adds a "Temporal Reasoning" bullet to the list of what changed.

| Benchmark | v2 → v3 | Tokens | p50 |
|---|---|---|---|
| LoCoMo | 71.4 → **92.5** | 7.0K | 0.88s |
| LongMemEval | 67.8 → **94.4** | 6.8K | 1.09s |
| BEAM (1M) | — → **64.1** | 6.7K | 1.00s |
| BEAM (10M) | — → **48.6** | 6.9K | 1.05s |

Methodology moved out of the repo in June 2026 (PR #5520): `evaluation/` is now a git submodule pointing at `mem0ai/memory-benchmarks`, which this study does not pin. The earlier in-repo harness compared against [LoCoMo](https://github.com/snap-research/locomo) paper baselines, ReadAgent, MemoryBank, MemGPT, A-Mem, LangMem, vanilla RAG, full-context, OpenAI's built-in memory, and Zep. The harness is open enough that StateBench-style independent verification is feasible — see [`Profile__StateBench.md`](./Profile__StateBench.md) for why you should want that.

## What's inside this submodule

| Path | What's there |
|---|---|
| `mem0/` | Core Python SDK (`mem0ai`, v2.2.1 at this pin): memory class, store/LLM/embedder/reranker abstractions |
| `mem0-ts/` | TypeScript SDK (v3.3.1) |
| `cli/{python,node}/` | CLI tools (`mem0-cli`, `@mem0/cli`) |
| `server/` | Self-hosted FastAPI + pgvector Postgres + dashboard stack |
| `integrations/` | All agent and framework plugins, consolidated in June 2026 (PR #5491): Claude Code, Codex, Cursor, Kimi, Antigravity, OpenCode, OpenClaw, pi, Hermes, DeepSeek Harness, Strands, Vercel AI SDK, n8n, Zapier, plus a shared `agent-plugin-core` runtime |
| `skills/` | Agent Skills for building with Mem0 (`mem0`, `mem0-cli`, `mem0-integrate`, `mem0-oss-to-platform`, ...) |
| `evaluation/` | Submodule pointing at `mem0ai/memory-benchmarks` |
| `examples/` | Sample projects and notebooks (the old `cookbooks/` folded in here) |
| `docs/` | Mintlify docs site |

`openmemory/` was removed (PR #6530). If you only read one file: `mem0/memory/main.py`. The whole behavioral surface is there (extraction, storage, search, scoring) in roughly 3,900 lines, about half of it the async twin.

## Mental model for using it well

- **Always pass a scope.** Even for a prototype. The day you bolt on multi-user is the day "I forgot to filter" becomes an incident.
- **Use `infer=False` when you already know.** "User confirmed email is alice@example.com" doesn't need an extraction LLM call — write it literally.
- **Don't expect supersession.** If correctness over time matters (a "what's my current address" workflow), build explicit `update` paths or read [`Profile__StateBench.md`](./Profile__StateBench.md) to understand the failure modes Mem0 is structurally exposed to.
- **Treat the entity store as the killer feature.** Pure vector recall plateaus quickly. The entity-link boost is what makes "tell me what we know about X" work without a real graph DB.
- **Use the self-hosted server before reaching for the platform.** The OSS dashboard is good enough for most teams; the hosted tier earns its keep when you need graph memory, user profiles, or audit-grade ops. Expect the OSS SDK to nudge you toward it; it now prints Platform notices.

## When NOT to reach for this

- **You need provenance and supersession** (compliance, agent-medical, agent-legal). Mem0's append-only-with-no-supersession will resurrect superseded facts. Use a state-based system (see Memgine in `Profile__StateBench.md`) or Neo (with its `superseded_by` chain).
- **You're a coding agent and want full immutable history.** That's Volt's job. Mem0 forgets what it didn't extract.
- **You want a graph-first model.** Use [[Profile__Graphiti]] (now pinned in this study); entity-link arrays are not a graph, and Mem0's own graph tier is Platform-only.
- **You're allergic to LLM-driven extraction.** Mem0 calls an LLM on every `add()`. For high-throughput pipelines this is a real cost line.
- **Local markdown notes are enough.** If your "memory" needs are "I want to look stuff up in my own notes," our own `context-vigilance` skill (markdown + wikilinks + git) is dramatically simpler and produces artifacts a human can read. Mem0 only earns its keep once you're routing agent calls.

## How this compares to the rest of the study

| Axis | Mem0 | Neo | Volt | StateBench (as a yardstick) |
|---|---|---|---|---|
| **Shape** | General memory layer | Code-reasoning memory + agent | Coding agent with built-in memory | Conformance benchmark |
| **Storage** | Vector + entity store + SQLite | Scoped JSON files w/ inline embeddings | Postgres DAG (immutable log + summary nodes) | N/A |
| **Write policy** | Append-only + entity link | Append-only + deterministic supersession (cosine > 0.85) | Verbatim append + async summary | Tests this directly |
| **Eviction** | None in OSS (optional `expiration_date` hides, never deletes) | Stale-prune + demotion + synthesis | Dolt evicts bindles; Upward never evicts | Penalizes wrong eviction |
| **Supersession** | Not modeled | First-class | First-class via immutable log | Measures resurrection rate |
| **Scope model** | user/agent/run/actor (required) | global/org/project/session (auto-detected) | conversation (per session) | Tests scope leak |
| **Recall** | Hybrid (semantic + BM25 + entity) | Semantic + confidence × success | Bindle expansion + `lcm_grep` | Tests must/must-not mention |
| **Operational story** | Library / server / hosted | Local CLI + MCP plugin | Local terminal + embedded Postgres | Harness / Hub Space |
| **Best fit** | Any agent that needs cross-session user memory | Code assistants that improve over time | Long-horizon coding sessions | Evaluating any of the above |

## How this compares to our own `context-vigilance` skill

Worth being explicit because the temptation to over-reach is real. The `context-vigilance` agent-skill is a *human-readable markdown discipline* — directory roles (`prompts/`, `specs/`, `blueprints/`, `reminders/`, `explorations/`, `issues/`), frontmatter, epoch.major.minor.patch versioning, wikilinks. It is **not** a memory layer; there is no embedding, no recall API, no scope-aware filter, no extraction. It assumes a human is in the loop and the artifacts are read by other humans (and by agents that grep + read markdown).

Mem0 is the right tool when: the consumer is an agent, the corpus is too large to keep in context, the scope axis is `user_id`/`agent_id` (not "this project's docs"), and the value of fuzzy recall outweighs the cost of LLM extraction. `context-vigilance` is the right tool when: the consumer is a human + an agent that can read the repo, the artifacts need to be reviewed in PRs, and the structure matters as much as the content.

They can coexist: `context-vigilance` for the durable project knowledge, Mem0 for ephemeral per-user/per-session memory the agent accumulates while operating.

## Changes since the previous pin (2026-04-29 → 2026-11-18)

Old pin `6d3486ca5671` (Python SDK v2.0.1 + 8 commits) → new pin `fff8a6472512` (v2.2.1 + 15 commits). 515 commits. Releases in range: Python SDK v2.0.2 through v2.2.1 (2026-09-25), TypeScript SDK ts-v3.0.3 through ts-v3.3.1, plus CLI, OpenClaw, OpenCode, pi-agent, Vercel AI, n8n, DeepSeek, and Strands plugin tags. License unchanged (Apache-2.0).

The core write and recall design is unchanged: single-pass additive extraction, entity linking, additive semantic + BM25 + entity scoring, required `user_id`/`agent_id`/`run_id`. What changed around it:

- **Expiration, not eviction.** `add()` and `update()` accept an optional `expiration_date`; expired memories are hidden from `search()` and `get_all()` unless `show_expired=True`, and never deleted (`mem0/memory/main.py:427-450, 769, 1369, 1684`).
- **Search explanations.** `search(explain=True)` returns per-result `score_details` (PR #5102).
- **Identity is immutable after creation.** Metadata on `add()` or `update()` can no longer set or overwrite `user_id`/`agent_id`/`run_id` (PRs #6656, #6278), and the session scope key now escapes its delimiters (PR #6892, `mem0/memory/main.py:407-419`).
- **Graph memory is officially Platform-only.** The docs now state it was removed from OSS (PR #6808); the self-hosted server compose has no graph service.
- **Benchmark claims revised and qualified.** README numbers rose to 92.5 (LoCoMo) and 94.4 (LongMemEval) and are now attributed to the managed platform with proprietary optimizations (`README.md:65`). The evaluation harness moved to the separate `mem0ai/memory-benchmarks` repo, mounted as the `evaluation/` submodule (PR #5520).
- **Repo reorganized.** All plugins moved under `integrations/` (PR #5491), `cookbooks/` folded into `examples/` (PR #5517), `openmemory/` removed (PR #6530), and a `skills/` directory of Agent Skills added.
- **Platform growth.** User Profiles v1 (PR #7340), Dream memory-synthesis endpoints, agent self-signup via `mem0 init --agent`, BYOK, and OSS-to-Platform migration tooling. None of these change the OSS library's behavior.
- **Robustness fixes** worth knowing if you run the OSS library: `delete_all` now paginates across vector-store pages (PR #6636), LLM extraction failures raise instead of returning `[]` (PR #5878), and an update racing a delete no longer resurrects the memory (PR #7787).

## One-line summary

> Mem0 is a hybrid vector + entity store with a CRUD-shaped agent-memory API, a single-pass write algorithm tuned for benchmark wins, three deployment modes that share one surface, and a deliberate refusal to model supersession — which makes it the right default for general agent memory and the wrong default for anything where state-correctness-over-time is the failure mode.
