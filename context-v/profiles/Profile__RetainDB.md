---
name: RetainDB Profile
slug: retaindb
upstream: https://github.com/retaindb/retaindb
package: "@retaindb/sdk, @retaindb/mcp, @retaindb/local (npm)"
license: Apache-2.0 (local + sdk + mcp) / BSL-1.1 (server)
maintainer: RetainDB
study: studies/memory-layers-for-agents
profile_path: studies/memory-layers-for-agents/retaindb
profile_kind: local-runtime + dual-licensed-server + hosted-cloud
date_created: 2026-05-26
site_uuid: b4d07a39-ed4f-4755-a346-503789666587
hex_code: 6r9bh1
date_authored_initial_draft: 2026-05-26
date_authored_current_draft: 2026-05-26
lede: >-
  Supersession as an actual indexed column — `validUntil`, `supersededBy` —
  rather than something the ranker is hoped to handle.
summary: >-
  Profile of RetainDB as pinned in the memory-layers-for-agents study. It is the
  study's second bi-temporal system (validity on records, where Graphiti puts it
  on edges) and its clearest worked example of typed memory driving decay
  policy. Covers the Prisma schema and its bi-temporal columns, the 13 memory
  types and 5 relation types with the two that trigger invalidation, the
  five-step supersession-aware write path and its stamped policy version, the
  weighted RRF hybrid retrieval with cross-encoder-then-LLM reranking, the
  no-Postgres single-process local runtime, three eviction mechanisms, and the
  scope column with its PROJECT-level option. Flag noted in the profile: the
  LICENSE-BSL file the README references is absent from the pinned snapshot, so
  verify server license terms upstream before any commercial planning.
publish: true
---

# RetainDB — Profile

A profile of RetainDB as it lives in this study (`studies/memory-layers-for-agents/retaindb/`). Cites pinned paths so you can jump to source rather than trust paraphrase. Read this alongside [[Profile__Mem0]] (the closest shape-twin — same library/server/cloud trifecta), [[Profile__Neo]] (the other system in this study that models supersession as a first-class column), and [[Profile__Graphiti]] (the other bi-temporal system, but with validity on edges instead of records).

## TL;DR

RetainDB is a memory layer for AI agents that takes the same library/server/cloud trifecta as [[Profile__Mem0]] and adds two structural bets Mem0 explicitly does not make: a **rich semantic-type enum** (13 memory types) and **bi-temporal validity columns** (`validFrom` / `validUntil` / `supersededBy`) on every memory record (`packages/server/prisma/schema.prisma:215-272`). The result is a memory store where "this fact replaced that fact" is a queryable column, not a recall-ranker hack.

It ships as a pnpm/turbo monorepo of four packages: a **single-process local runtime** (`@retaindb/local`, Apache-2.0) that runs on port 3111 with an atomic disk snapshot + append-only journal under `~/.retaindb/`; a **TypeScript SDK** (`@retaindb/sdk`, Apache-2.0) with adapters for Vercel AI SDK, LangChain, and LangGraph; an **MCP server** (`@retaindb/mcp`, Apache-2.0) exposing `memory_save`, `memory_smart_search`, `context`, `recall`, `handoff`, and friends; and a **Postgres + pgvector server** (`@retaindb/server`, **BSL-1.1**) with the full schema, hybrid retrieval, and ~20 ingestion connectors. The licensing split is the operative business-model wrinkle: the local/SDK/MCP path is permissive; running the server as a commercial hosted product requires a commercial license.

Retrieval is **hybrid by construction**. The server fuses vector + BM25 with weighted Reciprocal Rank Fusion (`packages/server/src/engine/retriever.ts:1429-1481`, default weights 0.7/0.3, RRF k=60), then optionally runs an LLM/cross-encoder reranker. The local runtime in `packages/local/src/cli.ts:578-620` fuses three signals (BM25 + vector + graph-via-concept-overlap) under RRF and layers `recency + decay + durability + reinforcement` heuristics on top. The marketing line *"BM25 + vector + graph retrieval with RRF and reranking"* (README.md:108) maps directly to that code.

If you want one sentence: **RetainDB is Mem0's shape with a richer type enum, bi-temporal supersession baked into Postgres columns, a local-first single-process runtime that does not require Postgres at all, and a dual-license posture that gives away the agent-side primitives while gating the commercial-hosting path.**

## Why this exists — the design bet

Two bets distinguish RetainDB from the rest of the field:

1. **Semantic memory typing is load-bearing.** Where Mem0 ships an optional `memory_type` string used mostly as metadata, RetainDB writes 13 canonical types into the type system itself (`packages/server/src/engine/memory/types.ts:6-19`) and the importance-decay policy keys off them (`packages/server/src/engine/memory/importance-decay.ts:34` — `factual` and `instruction` are flagged as `permanentTypes` that *never* decay). The bet: agents accumulate different kinds of memory with structurally different lifetimes, and the store should know which is which.
2. **Supersession is a column, not a vibe.** Every `Memory` row has `validFrom`, `validUntil`, `supersededBy`, `version`, and `isActive` (`schema.prisma:243-246`). When a new memory invalidates an old one — either because relation extraction returned a `RelationType` of `updates` or `contradicts`, or because the caller passed an explicit `supersedes_ids` (`packages/server/src/engine/memory/write.ts:951-962, 1021-1031`) — the old memory's `validUntil` is set to `now()`, `supersededBy` is set to the new id, `isActive` is flipped to false. The default search where-clause filters them out (`packages/server/src/engine/memory/search.ts:477` — `"validUntil" IS NULL OR "validUntil" > NOW()`). This is the direct contrast to [[Profile__Mem0]]'s "append-only with no supersession in OSS" posture.

A third operational bet matters at the org-level: **the local runtime intentionally has zero infrastructure dependencies** — no Postgres, no Redis, no Kafka, no Qdrant, no API keys (README.md:68). One npx command, one Node process, one JSON snapshot, one journal file. This is what makes "coding-agent memory on your laptop" tractable in a way the Mem0 self-hosted Docker compose stack is not.

## Storage topology

| Layer | Backend | Purpose | Code |
|---|---|---|---|
| **Local primary** | Atomic JSON snapshot + append-only JSONL journal | Memories, projects, shares, context snapshots — all under `~/.retaindb/` | `packages/local/src/cli.ts:56-70` |
| **Local embeddings** | hash-vector default; `@xenova/transformers` optional | Per-memory `embedding: number[]` on the in-memory record | `packages/local/src/cli.ts:72-75`, README.md:141 |
| **Server primary** | Postgres 16 + pgvector | `memories` (bi-temporal), `memory_relations`, `chunks` + `embeddings`, `entities` + `entity_relations`, `sessions` + `messages` | `schema.prisma:1-10, 215-349`, `docker-compose.yml:4-19` |
| **Server graph** | Same Postgres — `entity_relations` and `memory_relations` adjacency tables, no Neo4j | Cross-memory and cross-entity edges live in relational tables, not a graph DB | `schema.prisma:274-303, 332-349` |
| **Connectors** | Server-only | GitHub, Web/Sitemap, PDF, Notion, Confluence, Slack, Discord, arXiv, npm, PyPI, HuggingFace, Plain text | `packages/server/src/connectors/` (21 files) |

The architectural choice that contrasts with [[Profile__Mem0]] and [[Profile__Graphiti]]: **no separate graph database**. Graph signals come from the same relational store via the `MemoryRelation` and `EntityRelation` tables (`schema.prisma:274-291, 332-349`), plus a runtime-computed concept-overlap boost in the local engine (`cli.ts:642-650`). RetainDB is happy to call this a "memory graph" without ever standing up Neo4j.

## Schema of a memory record

From `packages/server/prisma/schema.prisma:215-272`:

```prisma
model Memory {
  id             String                 @id @default(uuid())
  projectId      String?
  orgId          String?                @default("default")
  userId         String?
  sessionId      String?
  agentId        String?
  taskId         String?

  memoryType     String                 @default("factual")
  content        String
  embedding      Unsupported("vector")?
  metadata       Json                   @default("{}")
  scope          String                 @default("USER")

  importance     Float                  @default(0.5)
  isActive       Boolean                @default(true)
  accessCount    Int                    @default(0)
  lastAccessedAt DateTime?
  recallCount    Int                    @default(0)
  expiresAt      DateTime?

  documentDate   DateTime?
  eventDate      DateTime?
  entityMentions String[]               @default([])
  confidence     Float                  @default(0.8)
  sourceChunkId  String?
  version        Int                    @default(1)
  validFrom      DateTime?              @default(now())
  validUntil     DateTime?
  supersededBy   String?
  // …indexes including @@index([validFrom, validUntil]) and @@index([supersededBy])
}
```

The canonical type enum (`packages/server/src/engine/memory/types.ts:6-19`):

```ts
export type MemoryType =
  | "factual"       // Objective facts: "User's name is John"
  | "preference"    // User preferences: "User prefers dark mode"
  | "event"         // Events: "User attended conference on Jan 15"
  | "relationship"  // Relationships: "Alex reports to Maria"
  | "opinion"       // Opinions: "User thinks React is great"
  | "goal"          // Goals: "User wants to learn Python"
  | "instruction"   // Instructions: "Always use formal tone"
  | "decision"      // Durable decisions: "The project standardizes on Bun"
  | "constraint"    // Constraints: "Deployment must stay on AWS Lambda"
  | "solution"      // Accepted fixes or solutions
  | "project_state" // Ongoing project state worth carrying forward
  | "correction"    // Correction that supersedes stale memory
  | "workflow";     // Reusable workflow or agent habit
```

The SDK `MemoryKind` (`packages/sdk/src/index.ts:207-220`) mirrors these 13 values. The relation enum (`types.ts:21-26`):

```ts
export type RelationType =
  | "updates"       // New memory supersedes old (State mutation)
  | "extends"       // Adds detail to existing memory (Refinement)
  | "derives"       // Inferred from other memories (Inference)
  | "contradicts"   // Conflicts with another memory
  | "supports";     // Provides evidence for another memory
```

Of these, **`updates` and `contradicts` are the two that trigger automatic invalidation** of the target memory (`packages/server/src/engine/memory/relations.ts:264-266` — `shouldInvalidateMemory`). The other three coexist with the older record.

Notable for what's **present** compared to [[Profile__Mem0]] §schema: `confidence`, `importance`, `version`, `validFrom`/`validUntil`/`supersededBy`, an `accessCount`+`recallCount`+`lastAccessedAt` triplet for reinforcement scoring, and `entityMentions` as a string array (in lieu of Mem0's separate entity-store collection).

## Write policy — supersession-aware, not append-only

The server write path (`packages/server/src/engine/memory/write.ts`) is structurally different from Mem0's single-pass extract. Each `add` runs:

1. **Relation detection** between the candidate memory and comparable memories already in scope (`write.ts:921-947` calls `detectRelations`).
2. **Explicit supersession merge** — if metadata includes `supersedes_ids`, those ids get an `"updates"` relation appended unconditionally (`write.ts:948-962`).
3. **Near-duplicate merge** — if no relation fired, a fuzzy-similarity check may merge into an existing record instead of creating a new one (`write.ts:964-986`).
4. **Version bump** — if any relations are invalidating, `nextVersion = max(superseded.version) + 1` (`write.ts:988-995`).
5. **Persistence + invalidation cascade** — create the new memory, then for each invalidating relation set the old memory's `validUntil = now()`, `supersededBy = newId`, `isActive = false` (`write.ts:1006-1031`).

A `MEMORY_WRITE_POLICY_VERSION = "memory_write_v2"` constant (`write.ts:14`) is stamped onto every memory-relation row's `metadata.policy_version`, which gives this system a property [[Profile__Mem0]] does not have: you can tell *which* write policy produced each relation, making policy migrations diffable.

Consolidation runs separately (`packages/server/src/engine/memory/consolidation.ts:226-289`): vector similarity > 0.95 clusters memories, an LLM merges the cluster into one canonical memory, and every member of the cluster is deactivated with `supersededBy` pointing at the merged record (`consolidation.ts:208-218`).

## Recall API

The library/SDK surface (`packages/sdk/src/retaindb.ts:82-110`) is fluent and CRUD-shaped:

```ts
const db = new RetainDB({ apiKey, baseUrl, project });
await db.user(userId).remember(content);                          // write
const { context } = await db.user(userId).getContext(query);      // read
const items = await db.user(userId).searchMemory(query);          // search
await db.user(userId).forget(memoryId);                           // delete
// One-shot retrieve → generate → store:
const turn = await db.user(userId).runTurn({ messages, generate });
```

The REST surface (`packages/local/src/cli.ts:971-1130`, `packages/server/src/api/`) is wider:

| Verb | Path | What |
|---|---|---|
| POST | `/v1/memory` | Store a memory |
| POST | `/v1/memory/bulk` | Batch store |
| POST | `/v1/memory/search` | Hybrid search with filters |
| POST | `/v1/context/query` | Packed-context retrieval (memories + chunks + graph) |
| POST | `/v1/context/pack` | Token-budgeted coding context pack with delta hash |
| POST | `/v1/context/delta` | Changed-only context since previous hash |
| POST | `/v1/context/compress-output` | Compress test/build/tool noise |
| POST | `/v1/context/code-map` | Compact file + symbol map |
| POST | `/v1/memory/ingest/session` | Ingest messages + work events |
| GET  | `/v1/memory/session/:id`, `/v1/memory/profile/:userId` | Session / profile listings |

Hybrid retrieval in the server (`packages/server/src/engine/retriever.ts:404-507`) runs four signal streams in parallel, then fuses:

```ts
// Vector + BM25 + memory + (optional) graph traversal
const vectorResults = await vectorSearch(...);                     // pgvector cosine
if (hybridSearch) bm25Results = await fullTextSearch(...);         // Postgres FTS
if (includeMemories) memoryResults = await memorySearch(...);      // bi-temporal scoped
if (includeGraph) graphResults = await graphSearch(...);           // entity-relation walk
// Then:
allResults = reciprocalRankFusion(allResults, vectorWeight=0.7, bm25Weight=0.3, k=60);
allResults = allResults.filter(r => r.score >= threshold);
if (rerank) allResults = await rerankResults(query, allResults, ...);
```

The `reciprocalRankFusion` (`retriever.ts:1429-1481`) is a weighted RRF: `rrfScore = weight / (k + rank + 1)` per stream, summed across streams, with results re-tagged `source: "hybrid"` when they appeared in both vector and BM25 lists. The reranker (`retriever.ts:1493+`) uses a cross-encoder fast path with an LLM fallback when confidence is low — quoted internal target is "93% accuracy at 260ms avg, only 20% use LLM" (`retriever.ts:1488-1492`).

The local engine's equivalent (`packages/local/src/cli.ts:540-620`) computes BM25 + cosine + graph-via-concept-overlap, ranks each signal, sums `1 / (RRF_K + rank)` across them, then adds heuristic scores: `score = rrf * 100 + importance + confidence + recency + decay + durability + reinforcement`. The `durability` term keys directly off `memory_type` (`cli.ts:575` — `semantic: 0.7, procedural: 0.6, correction: 0.5, session_summary: 0.35, else: 0`), which is what the type enum buys you operationally.

## Eviction & compaction

Three mechanisms, each keyed off a different signal:

| Mechanism | Trigger | Effect | Code |
|---|---|---|---|
| **Supersession** | `updates`/`contradicts` relation OR explicit `supersedes_ids` | `validUntil = now()`, `isActive = false`, `supersededBy = newId` | `write.ts:1021-1031` |
| **Consolidation** | Cosine similarity > 0.95 inside a cluster | LLM merges N memories → 1 canonical; all members deactivated | `consolidation.ts:155-218, 226-289` |
| **Importance decay** | Time + access pattern, gated by `memoryType` | Exponential decay with 30-day half-life, floor 0.1; `factual` and `instruction` *never* decay | `importance-decay.ts:28-35, 40-90` |

This is materially richer than [[Profile__Mem0]]'s "none in OSS" eviction story (Mem0 §Eviction). The trade-off: every one of these mechanisms touches `db.memory.update` — RetainDB pays for supersession in write amplification, but never asks the recall ranker to silently bury a stale fact.

## Scopes & namespacing

Five named scopes via `Memory.scope` (`schema.prisma:228`, default `"USER"`) — `USER | SESSION | PROJECT | AGENT | TASK | DOCUMENT` (`packages/server/src/engine/memory/types.ts:28-35`). The recall where-clause builds identity filters per scope (`search.ts:483-491`):

```ts
const identityFilters = [Prisma.sql`"scope" = 'PROJECT'`];
if (userId)    identityFilters.push(sql`("scope" = 'USER' AND "userId" = ${userId})`);
if (sessionId) identityFilters.push(sql`("scope" = 'SESSION' AND "sessionId" = ${sessionId})`);
if (agentId)   identityFilters.push(sql`("scope" = 'AGENT' AND "agentId" = ${agentId})`);
if (taskId)    identityFilters.push(sql`("scope" = 'TASK' AND "taskId" = ${taskId})`);
// Always `OR`-joined, then AND-ed with validUntil and isActive predicates.
```

Compared to [[Profile__Mem0]]'s tri-axis `user_id`/`agent_id`/`run_id` (all optional, at least one required), RetainDB adds a `task_id` axis and a `scope` discriminator that lets a memory deliberately live at the *project* level (not tied to any user). This matches the coding-agent use case: "the project standardizes on Postgres" is project-scope; "this user prefers concise answers" is user-scope. Mem0 collapses both into per-record metadata; RetainDB makes scope a row-level column.

## Operational story — three modes, three licenses

| Mode | Process | Storage | License | When |
|---|---|---|---|---|
| **Local** (`@retaindb/local`) | One Node process on `:3111` + viewer on `:3113` | JSON snapshot + JSONL journal under `~/.retaindb/` | Apache-2.0 | Coding-agent memory on a single machine |
| **Self-hosted server** (`@retaindb/server`) | `docker compose up` — Hono + Prisma + Postgres 16 + pgvector | Postgres tables per `schema.prisma` | **BSL-1.1** | Team / product backend; full connector + relation graph |
| **Hosted cloud** (`api.retaindb.com`) | Managed | Managed | Commercial | Multi-tenant auth, dashboards, billing, lifecycle email |

The MCP layer (`@retaindb/mcp`) is shared infrastructure — the same MCP package speaks to all three deployment modes by varying `RETAINDB_BASE_URL`. Tools exposed (`packages/mcp/src/server.ts:203-957`): `memory_save`, `memory_smart_search`, `memory_sessions`, `context`, `agent_event`, `recall`, `session_history`, `handoff`, `resume_handoff`, plus a longer tail for replay and observation. The local runtime ships first-class `connect all` setup for Codex, Claude Code, and OpenCode (README.md:56-60).

The BSL-1.1 split is the operative business posture. From `README.md:494-501`: *"`packages/server`: Business Source License 1.1 — Self-hosting is free. Building a hosted service on top of the server requires a commercial license."* Note: the `LICENSE-BSL` file the README references is not present in the pinned submodule snapshot (only the Apache-2.0 `LICENSE` at the root). The server package's `package.json` has no `license` field at all, so the actual license terms must be read from the upstream repo before any commercial planning.

## What's inside this submodule

| Path | What's there | License |
|---|---|---|
| `packages/local/` | One-process local runtime — Hono server, journal, viewer, RRF retrieval, connect-all CLI; one 1,965-line `cli.ts` | Apache-2.0 |
| `packages/sdk/` | TypeScript SDK with adapters for Vercel AI SDK, LangChain (`adapters/langgraph.ts`), tool-call shim, `RetainDB` fluent class, `whisper-agent` for background extraction | Apache-2.0 |
| `packages/mcp/` | MCP server exposing ~19 tools; speaks to local or server via `RETAINDB_BASE_URL` | Apache-2.0 |
| `packages/server/` | Postgres + pgvector backend, Prisma schema (703 lines), Hono API, 21 connectors, full memory engine (`engine/memory/` — write, search, relations, consolidation, temporal, decay, extractor) | **BSL-1.1** (per README; LICENSE-BSL file missing from snapshot) |
| `docker-compose.yml` | pgvector/pgvector:pg16 + server | — |
| `pnpm-workspace.yaml`, `turbo.json` | Workspace orchestration | — |

If you only read one file: `packages/server/prisma/schema.prisma` — the bi-temporal columns, the relation table, the entity store, and the index choices (`@@index([validFrom, validUntil])`, `@@index([supersededBy])`) tell the architectural story in 700 lines.

## Mental model for using it well

- **Type your memories.** The 13-type enum is the lever that makes `durability`, decay-immunity, and rerank-bonuses do useful work. A memory you wrote as `"factual"` will never decay; one you wrote as `"event"` will. Choosing badly is choosing wrong on a load-bearing axis.
- **Treat `correction` as the supersession verb.** If the user says "actually it's X, not Y", write `memory_type: "correction"` and either let relation-detection find the target or pass `supersedes_ids` explicitly (`write.ts:951-962`). Mem0's "two memories, both true-looking" failure mode is exactly what this is designed to prevent.
- **Start local, graduate when you need connectors or multi-tenancy.** The local runtime has no Postgres dependency and no API key requirement. It is structurally the same memory model, minus the connectors and the relational graph tables. Most prototypes never need to graduate.
- **Use the MCP package for editor agents.** It's Apache-2.0, it ships `connect all` snippets for the three big coding agents, and it speaks to local or server identically.
- **If you're building a product, read the server license before you scale.** Self-hosting is free; building a hosted memory service on top of `packages/server` is not.

## When NOT to reach for this

- **You are building a commercial hosted memory product on top of the server.** The server is BSL-1.1. You will need a commercial license from RetainDB. This is the single most important read-the-license-first situation in this study. If permissive licensing is non-negotiable, look at [[Profile__Mem0]] (full Apache-2.0 including the self-hosted server) or [[Profile__Letta]].
- **You need true graph-database semantics** — multi-hop traversal, Cypher queries, dedicated graph algorithms. RetainDB's "graph" is a relational adjacency table. [[Profile__Graphiti]] is the right tool when the graph *is* the model.
- **Your memory needs are flat and metadata-driven.** If you don't need supersession, bi-temporal queries, or the type enum, [[Profile__Mem0]]'s smaller surface and benchmark-tuned single-pass write is faster and simpler.
- **You are evaluating against LoCoMo/LongMemEval head-to-head.** RetainDB ships internal claims ("76.69% on LongMemEval temporal reasoning" — `temporal.ts:4`) but no public evaluation harness in this snapshot. [[Profile__Mem0]] publishes a reproducible `evaluation/` directory; RetainDB does not.
- **Local markdown notes are enough.** The Lossless `context-vigilance` skill (markdown + wikilinks + git) remains the right answer when a human is in the loop and the artifacts need to be reviewable in PRs. RetainDB earns its keep only once the consumer is an agent and the corpus is too big to keep in context.

## How this compares to the rest of the study

| Axis | RetainDB | [[Profile__Mem0]] | [[Profile__Neo]] | [[Profile__Graphiti]] |
|---|---|---|---|---|
| **Shape** | Local runtime + server + cloud | Library + server + cloud | Code-reasoning memory + agent | Graph-native temporal memory |
| **Primary storage** | Postgres + pgvector (server); JSON+journal (local) | Vector DB + SQLite (Qdrant default) | Scoped JSON files w/ embeddings | Neo4j (or FalkorDB) with bi-temporal edges |
| **Memory types** | 13 (factual/preference/decision/constraint/correction/…) | 1 string field, not enforced | Single "fact" type | N/A (graph edges, not typed memories) |
| **Supersession** | First-class column (`validUntil` + `supersededBy`) | Not modeled in OSS | First-class via `superseded_by` pointer | First-class via bi-temporal edges |
| **Bi-temporal** | Yes, on memory records | No | No | Yes, on graph edges |
| **Recall** | BM25 + vector + graph + RRF (k=60) + cross-encoder/LLM rerank | Hybrid (semantic + BM25 + entity boost), additive fusion | Semantic + confidence × success | Graph traversal + edge-validity filter |
| **Eviction** | Supersession + consolidation + type-keyed decay | None in OSS | Stale-prune + demotion + synthesis | Edge `valid_to` |
| **Scope model** | scope column (USER/SESSION/PROJECT/AGENT/TASK/DOCUMENT) | user/agent/run/actor (required) | global/org/project/session (auto) | Per-graph-namespace |
| **License** | **Apache-2.0 + BSL-1.1 (server)** | Apache-2.0 | Apache-2.0 | Apache-2.0 |
| **MCP** | First-class, Apache-2.0 package | Editor plugins | MCP plugin | Indirect via Zep |
| **Best fit** | Coding-agent memory + product memory where supersession matters | General agent memory at benchmark-tuned latency | Code assistants that improve over time | "When did X change?" graph queries |

## One-line summary

> RetainDB is the [[Profile__Mem0]]-shaped memory layer that *also* models supersession — 13 typed memory categories with `validFrom`/`validUntil`/`supersededBy` columns, a one-process local runtime that needs no Postgres, BM25 + vector + graph retrieval fused with weighted RRF, and a permissive license on every package the agent touches except the server itself, which is BSL-1.1 and requires a commercial license to host as a service.
