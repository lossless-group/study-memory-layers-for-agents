---
name: Supermemory Profile
slug: supermemory
upstream: https://github.com/supermemoryai/supermemory
package: supermemory (npm), supermemory (PyPI), @supermemory/tools (npm), supermemory_agent_framework (PyPI), supermemory_openai (PyPI), supermemory_pipecat (PyPI), supermemory_cartesia (PyPI)
license: MIT
maintainer: Supermemory (Dhravya Shah et al.)
study: studies/memory-layers-for-agents
profile_path: studies/memory-layers-for-agents/supermemory
profile_kind: hosted-platform + thin-clients + SDKs + connectors + MCP-server
date_created: 2026-05-26
site_uuid: 85801db5-f917-4e49-8f1a-c1c5edf9fbe5
hex_code: wuyi0o
date_authored_initial_draft: 2026-05-26
date_authored_current_draft: 2026-11-23
lede: >-
  The most polished client surface in the study, wrapped around an engine that
  is not in the repo — every SDK points at a hosted API or a closed
  self-hosted binary.
summary: >-
  Profile of Supermemory as pinned in the memory-layers-for-agents study. Its
  role here is as the verifiability case: the entry with the loudest benchmark
  claims and the least inspectable engine. Covers what the OSS repo actually
  ships (Cloudflare MCP server, SDK integrations and framework adapters, the
  Zod validation schemas, a memory-graph visualizer, the docs; the consumer app
  is now a redirect shell), the MemoryEntry schema with its version chain and
  forget metadata, the ingest endpoints, the profile() call as the distinctive
  recall primitive, containerTag scoping (renamed namespace in the v5 API) and
  its unsafe default, the closed self-hosted binary, the connector
  discrepancy between the README and the validated provider enum, and an honest
  reading of three mutually incommensurable "#1" claims. The recommended posture
  is in the mental-model section: verify forgetting and supersession in your own
  eval rather than on the README's word.
publish: true
title: Supermemory — Profile
date_modified: 2026-11-23
authors:
  - Michael Staton
status: Reference
type: Profiles
---

# Supermemory — Profile

A profile of Supermemory as it lives in this study (`studies/memory-layers-for-agents/supermemory/`). Cites pinned paths so you can jump to source rather than trust paraphrase. Read alongside [`Profile__Mem0.md`](./Profile__Mem0.md) — Supermemory is the closest peer in scope and is the entry making the strongest benchmark claims in the field — and [`Profile__MemPalace.md`](./Profile__MemPalace.md), which makes a structurally opposite extraction-vs-verbatim bet on the same benchmark.

## TL;DR

Supermemory positions itself as the **state-of-the-art memory and context engine for AI**, and pins the claim on three rankings (`README.md:32, 39`, `README.md:371-373`): **#1 on LongMemEval, #1 on LoCoMo, #1 on ConvoMem** — the three headline harnesses the rest of the study competes on. At the original pin the README gave LongMemEval as 81.6%; at the current pin that figure is gone and the README instead reports 95% Recall@15 on LongMemEval with ~720 tokens of added context (`README.md:375`), all vendor-reported. The repo was one of the most active OSS entries in the study (1,637 commits at the original pin, 372 more since), MIT-licensed.

The bet is **universal RAG + memory in one engine**: fact extraction from conversations, auto-maintained user profiles (static + dynamic), hybrid semantic search that interleaves user memories with knowledge-base chunks, multi-modal extractors (PDF/OCR/transcription/AST-aware code chunking), and real external connectors that webhook-sync Google Drive / Gmail / Notion / OneDrive / GitHub into the same store (`README.md:45-52`, `README.md:294`).

The most consequential thing to know reading the repo: **the memory engine is not in it.** Every SDK and surface in `supermemory/` defaults to `https://api.supermemory.ai` (`apps/mcp/src/server/index.ts:21`, `apps/mcp/src/server/client/index.ts:251`, `packages/lib/api.ts:447`, `packages/tools/src/shared/context.ts:11`). The extraction prompt, the supersession reconciliation logic, the hybrid-search ranker, the eviction policy — all closed-source. Since mid-2026 there is also a self-hosted server, `supermemory-server` ("Supermemory local"), but it ships as a downloadable binary that the docs say "is built from a separate, non-public codebase" (`apps/docs/self-hosting/overview.mdx:21`). So you can now run the engine yourself; you still cannot read it. What is in the repo is (a) a redirect shell for the old consumer app (`apps/web/README.md`), (b) the MCP server (a Cloudflare Worker with Durable Objects that proxies to the API), (c) the SDK integrations (TS framework adapters in `@supermemory/tools` for Vercel AI SDK / Mastra / OpenAI Agents / VoltAgent, plus five Python wrappers), (d) the Zod schemas the hosted API validates against, (e) a Claude Code skill, (f) a force-directed memory-graph visualizer, and (g) the Mintlify docs, including the v5 API reference.

**One sentence:** *Supermemory is a closed-source memory engine, hosted or run as a closed binary, whose OSS repo ships an unusually complete thin-client surface (TS + Python SDK integrations, framework adapters, MCP, Claude Code skill), making it the most operationally polished entry in the study — and the hardest to verify, because the engine that produces the benchmark numbers is not in the codebase you can read.*

## Why this exists — the design bet

The thesis is laid out plainly in `README.md:39-52` and elaborated in `skills/supermemory/references/architecture.md:5-36`: agents should not be choosing between "RAG" (stateless document retrieval) and "memory" (per-user state). The bet is that there is one *living knowledge graph* — extracted facts, ingested documents, connector-synced content — that the agent queries with one API, getting both knowledge-base hits and personalized context in the same response (`README.md:412`).

Five design moves follow:

1. **Memory is extracted, not stored verbatim.** Conversations go through the engine and come back as discrete `MemoryEntry` rows with parent/version chains. This is the direct opposition to [[Profile__MemPalace]] (verbatim chunks, no extraction) and aligned with [[Profile__Mem0]] (single-pass extraction).
2. **User profile is the recall primitive, not just memories.** The headline `client.profile()` call returns `{ profile.static, profile.dynamic, searchResults }` in one round-trip, ~50ms (`README.md:213-216`, `apps/mcp/src/server/tools/get-profile.ts`). Most peers force you to compose this from `search` calls; Supermemory bakes it in.
3. **Hybrid search interleaves RAG and memory by default.** `searchMode: "hybrid"` is the default; `"memories"` is opt-in (`README.md:265, 304`); the MCP server's own search pins `searchMode: "memories"` (`apps/mcp/src/server/client/index.ts:308-316`). The marketing pitch — "Memory is not RAG" (`README.md:412`) — is contradicted by the architectural pitch, which is "RAG and memory together."
4. **Supersession and forgetting are first-class in the schema.** `MemoryEntry` carries `version`, `parentMemoryId`, `rootMemoryId`, `isLatest`, `isForgotten`, `forgetAfter`, `forgetReason` (`packages/validation/schemas.ts:242-278`). Whether the engine *uses* these fields well is unverifiable from the OSS code — only the schema is.
5. **Connectors are part of the engine, not a separate product.** Real-time webhook sync from five external SaaS providers feeds the same store as the chat memories (`README.md:52, 294`; the web dashboard that held the connector settings was removed, see below).

## Storage topology

The on-disk shape is mostly **inferred from the Zod schemas and the documents API response**, not from engine code. Treat as authoritative for the wire contract; treat as illustrative for what the engine actually does.

| Layer | Backend (per `CLAUDE.md` + schemas) | Purpose | Code |
|---|---|---|---|
| Documents | Cloudflare Hyperdrive (Postgres on the wire) | Top-level ingest unit — file, URL, conversation, tweet | `packages/validation/schemas.ts:61-101` |
| Chunks | Postgres + vector column | Semantic chunks of a document with embedding + matryoshka embedding | `packages/validation/schemas.ts:106-124` |
| MemoryEntry | Postgres + vector column | Extracted facts, versioned, with parent chain and forget metadata | `packages/validation/schemas.ts:242-278` |
| Spaces | Postgres | Container-tag-backed isolation unit; carries a knowledge-base index | `packages/validation/schemas.ts:218-237` |
| Connections | Postgres | OAuth state + per-connection metadata for the five connectors | `packages/validation/schemas.ts:126-167` |
| Memory↔Document join | Postgres | `relevanceScore`, `addedAt`, multi-source attribution | `packages/validation/schemas.ts:287-294` |
| Embedding model | Cloudflare Workers AI (per `CLAUDE.md:75`) | "Vector embedding generation using Cloudflare AI" | `CLAUDE.md:70-83` |
| Index structure | HNSW (per skill reference) | Sub-millisecond similarity lookups | `skills/supermemory/references/architecture.md:359-364` |

The skill reference (`architecture.md:452-484`) draws this as "Vectors (HNSW) + Relationships (Graph)" but no graph DB is named anywhere in the OSS code I read. Relations live as columns on `MemoryEntry` (`memoryRelations: z.record(MemoryRelationEnum).default({})`, line 256), not as edges in Neo4j or similar — which is closer to [[Profile__Mem0]]'s "entity-link arrays as implicit graph" than to a real graph store like [[Profile__Graphiti]].

## Schema of a memory record

The `MemoryEntry` schema (`packages/validation/schemas.ts:242-278`) is the most informative single file in the repo, because it tells you what fields the engine writes to even though you cannot see the engine.

```typescript
{
  id: string,
  memory: string,                    // extracted fact text
  spaceId: string,                   // container scope
  orgId: string,
  userId: string?,

  // Versioning — supersession is first-class
  version: number,                   // monotonic
  isLatest: boolean,
  parentMemoryId: string?,           // points at the superseded entry
  rootMemoryId: string?,             // points at the v1 of the chain

  // Relations to other memories (NOT a separate edge table)
  memoryRelations: Record<id, "updates" | "extends" | "derives">,

  // Provenance — many docs can source one memory
  sourceCount: number,               // mirrored via MemoryDocumentSource join

  // Status flags — forgetting is structurally modeled
  isInference: boolean,              // engine-derived vs literal
  isStatic: boolean,                 // stable preference vs episodic
  isForgotten: boolean,
  forgetAfter: Date?,                // TTL
  forgetReason: string?,             // why this memory expired

  // Two embedding columns — main + migration-ready
  memoryEmbedding: number[]?,
  memoryEmbeddingModel: string?,
  memoryEmbeddingNew: number[]?,     // dual-write for embedder upgrades
  memoryEmbeddingNewModel: string?,

  metadata: Record<string, unknown>?,
  createdAt: Date, updatedAt: Date,
}
```

Compare to [[Profile__Mem0]] `MemoryItem` (which has `hash`, `text_lemmatized` for BM25, no version chain, no forget metadata) and [[Profile__MemPalace]] drawer (verbatim chunk, deterministic ID, no version, no provenance chain). Supermemory's schema is the richest in the study after [[Profile__Neo]]. **Whether the engine actually populates these fields with the discipline the schema implies is the unverifiable question.**

The version-chain index logic in the OSS visualizer (`packages/memory-graph/src/canvas/version-chain.ts:39-78`) walks `parentMemoryId` backward to the root and forward through the first-child branch — confirming the chain is linear in practice, not branching. A "user said X then changed to Y then changed to Z" produces v1→v2→v3, with `isLatest=true` only on v3.

## Write policy — extraction in the cloud, conversations on the wire

The write surface ships **two distinct ingest endpoints**, both POSTed to the hosted API:

1. **`POST /v3/documents`** (v5: `/ns/{namespace}/document`) — unstructured content (URL, PDF, image, video, text). Triggers the six-stage `IngestContentWorkflow` (queued → extracting → chunking → embedding → indexing → done) per `CLAUDE.md:70-76` and `skills/supermemory/references/architecture.md:37-168`. Documents are the unit of upload; memories are the *output* of extraction.
2. **`POST /v4/conversations`** — structured chat. At the original pin the adapters POSTed here directly (`packages/tools/src/conversations-client.ts`); at the current pin they go through the v5 `supermemory` SDK client (`packages/tools/src/conversations-client.ts:1-5`). Takes `{ conversationId, messages, containerTags, metadata, entityContext }`. The endpoint comment claims it "supports structured messages with smart diffing and append detection on the backend" (original-pin `conversations-client.ts:7`) — i.e., the engine handles incremental updates of the same conversation. Cannot be verified from this repo.

The framework adapters write via `/v4/conversations`. The Vercel AI SDK wrapper saves the conversation **after** every assistant response (`saveMemoryAfterResponse`, `packages/tools/src/vercel/middleware.ts:156-180`), with `addMemory: "always"` as the default (`packages/tools/src/vercel/index.ts:160`). v5 also adds a `dreaming` ingest option: `"instant"` extracts immediately, the default `"dynamic"` batches extraction, so a fresh namespace can show no memories for several minutes (`apps/docs/v5/api-reference/ingest.mdx:39`). This is closer to [[Profile__Letta]]'s self-editing pattern than to [[Profile__Mem0]]'s "extract per turn" — Supermemory ships the full conversation each round and lets the engine decide what's new.

**There is no fact-extraction prompt anywhere in this repo.** I grepped for `MEMORY_EXTRACTION_PROMPT`, `extraction_prompt`, `fact_extraction`, `systemPrompt` — nothing. Compare to [[Profile__Mem0]] where the entire write algorithm is one inspectable file (`mem0/configs/prompts.py:468`). Supermemory's extraction policy is unreadable from outside.

What the schema reveals about the engine's write policy:

- It produces inferred memories (`isInference: true`) distinct from literal ones.
- It distinguishes stable preferences from episodic context (`isStatic` flag).
- It writes new memory rows rather than mutating in place on contradiction — `parentMemoryId` + `version` pattern; the old row stays, `isLatest` flips false.
- It accumulates multi-source attribution via the `MemoryDocumentSource` join table with a per-source `relevanceScore` (`schemas.ts:287-294`).

## Recall API

Three surfaces, increasing specificity:

```typescript
// Primary: profile + optional search in one round-trip
const { profile, searchResults } = await client.profile({
  containerTag: "user_123",
  q: "programming style",       // optional
});
// profile.static  → ["Senior engineer at Acme", "Prefers Vim"]
// profile.dynamic → ["Working on auth migration"]
// searchResults   → ranked memories matching q (only if q provided)

// Hybrid search across memories + RAG documents
const results = await client.search.memories({
  q: "how do I deploy?",
  containerTag: "user_123",
  searchMode: "hybrid",          // | "memories" — default is hybrid
  threshold: 0.6,                // optional similarity gate
  limit: 10,
});

// Document-level search with metadata filters
const docs = await client.search.documents({ ... });
```

Source at the original pin: `apps/mcp/src/client.ts` (memory search and profile) and `packages/tools/src/shared/memory-client.ts` (the `/v4/profile` POST). At the current pin the MCP client wraps the v5 SDK (`client.search(namespace, …)` and `client.profile(namespace)`, `apps/mcp/src/server/client/index.ts:308-345`), and the adapters call the same two methods (`packages/tools/src/shared/memory-client.ts:30-60`). v5 moves the scope into the path (`/ns/{namespace}/search`, `/ns/{namespace}/profile`) and the code samples above are the README's v4-style shape (`README.md:203-282`).

Filter operators on `search.documents` mirror [[Profile__Mem0]]'s richer set: `AND`/`OR` boolean groups with `numericOperator` (`>`/`>=`/`<`/`<=`/`=`) over string/number/boolean values (`packages/validation/api.ts:251-323`). This is the metadata filter shape the engine accepts on the wire.

The retrieval mechanism in the skill docs (`skills/supermemory/references/architecture.md:204-244`) describes the search path as **embed → vector lookup → threshold filter → relationship expansion (follow `extends`/`derives`/`updates` from candidate memories) → rank** by similarity + recency + static-priority + relationship-strength. Again — the doc is marketing, not source. The actual implementation is closed.

## Eviction & compaction — schema says yes, implementation unverifiable

The `MemoryEntry` schema models forgetting end-to-end (`schemas.ts:262-266`): `isForgotten`, `forgetAfter` (TTL date), `forgetReason` (free-text). At the original pin the MCP server exposed a `memory(action: "forget")` tool that forgot by content with an exact-match-then-semantic-fallback at similarity ≥ 0.85. At the current pin `forget_memory` is a separate tool that forgets by memory ID only (`apps/mcp/src/server/client/index.ts:284-305`, `apps/mcp/src/server/tools/forget-memory.ts`; commit `ccf04858`). The marketing claim is bolder (`README.md:49`):

> Extracts facts from conversations. Handles **temporal changes, contradictions, and automatic forgetting**.

…and (`README.md:414`):

> Automatic forgetting. Supermemory knows when memories become irrelevant. Temporary facts ("I have an exam tomorrow") expire after the date passes. Contradictions are resolved automatically. Noise never becomes permanent memory.

**Verdict on whether this is real or aspirational:** the *schema* is real (the fields exist; the documents API returns them; the visualizer renders chains). The *policy* — when the engine decides to set `forgetAfter`, when contradictions trigger a new version vs. an explicit forget, what counts as "noise" — is invisible. Compare to [[Profile__Mem0]] (no eviction at all in OSS), [[Profile__Volt]] (deterministic three-level escalation that you can read), [[Profile__Neo]] (`prune_stale_facts` / `demote_unhelpful_facts` as readable Python). Supermemory's forgetting is the only one in the study where the *user can't verify it works*. That is structurally a research integrity concern for any benchmark that claims it.

## Scopes & namespacing

The unit was **`containerTag`** (`packages/validation/api.ts:122`) — a free-form string, max 128 chars. The v5 API renames it **namespace** and moves it into the path, one namespace per request (`/ns/{namespace}/…`); existing tag values stay valid (`apps/docs/snippets/api-v5-namespaces.mdx`). By convention:

- A user ID is the typical "root" container (the MCP OAuth flow attaches one).
- Additional tags scope memories to projects (the MCP server now exposes them as namespaces and spaces: `list_spaces`, a `namespaces` resource, and a per-account active space held in a Durable Object, `apps/mcp/src/server/resources/namespaces.ts`, `apps/mcp/README.md:11-18`).
- A memory can carry multiple `containerTags` (`MemorySchema.containerTags: z.array(z.string())`) — i.e., scopes overlap by design.

Layered above container tags are **Spaces** (`schemas.ts:218-237`) with a separate access-control table (`SpacesToMembersSchema:299-307`) carrying owner/admin/editor/viewer roles. Spaces are the multi-user share primitive; container tags are the single-user partition.

Above Spaces is **Organization** (Better Auth-managed per `CLAUDE.md:36, 56`). The `MemoryEntry` carries `orgId` + `userId` at the row level; tenancy is enforced at the engine, not the client.

This is more elaborate than [[Profile__Mem0]] (user/agent/run/actor — required at every call) and more elaborate than [[Profile__MemPalace]] (wing/room/agent/diary). The trade-off at the original pin: scope was *optional* at the API surface (defaulted to `sm_project_default` when omitted), so the same "I forgot to scope and leaked Alice's memories to Bob" failure mode Mem0 designs against was **possible here**. v5 narrows this: the SDK takes the namespace as a required argument (`packages/tools/src/vercel/index.ts:20, 101`). The MCP server still falls back to `sm_project_default` when no space is named (`apps/mcp/src/server/client/index.ts:22, 261`).

## Operational story

Tiers in a different shape from Mem0's library/server/hosted split — because **the library and server are not in the OSS repo**.

1. **Hosted API** — `https://api.supermemory.ai`. At the original pin: v3 for documents, v4 for conversations + profile + search. At the current pin: the namespace-scoped v5 API, with v3/v4 kept as legacy (`apps/docs/migration/api-v5.mdx`). Authentication via Bearer token from `console.supermemory.ai`.
   **Self-hosted binary** — `supermemory-server` / `npx supermemory local`, released under `server-v*` tags (latest `server-v0.0.9`, the first that speaks v5). Embedded graph engine, local `bge-base-en-v1.5` embeddings, bring-your-own LLM including Ollama; telemetry on by default, off with `SUPERMEMORY_DISABLE_TELEMETRY=1` (`README.md:314-361`, `apps/docs/self-hosting/overview.mdx`). The binary's source is not public.
2. **Self-hosted thin clients** — what you can run from this repo:
   - `apps/web/` — at the original pin the Next.js consumer app (Nova); since #1651 a redirect shell that forwards `app.supermemory.ai` to the console (`apps/web/README.md`). The old app is tagged `archive/web-2026-09`.
   - `apps/mcp/` — Cloudflare Worker MCP server with Durable Objects for legacy protocol state and the active space, the `https://mcp.supermemory.ai/mcp` endpoint open-sourced for review (`README.md:135`, `apps/mcp/src/server/index.ts`).
   - `apps/browser-extension/` and `apps/raycast-extension/` — present at the original pin, both removed since (#1651, #1818).
   - `apps/sdk-playground/` — interactive SDK chat playground (new).
   - `apps/memory-graph-playground/` — visualizer demo.
3. **SDKs** (npm + PyPI):
   - **TS:** `supermemory` (SDK on npm, 5.x for v5; this repo *consumes* it), `@supermemory/tools` (the framework adapters in this repo, 3.2.0 at the current pin, on v5). `@supermemory/ai-sdk` was retired in favor of `@supermemory/tools` 3.0 (#1773).
   - **Python:** `supermemory` (SDK on PyPI), `supermemory_agent_framework`, `supermemory_openai`, `supermemory_pipecat`, `supermemory_cartesia`, and a new LiveKit Agents wrapper — each a separate framework adapter package shipped from `packages/*-python/`.
4. **Editor plugins** — separate repos referenced from the README (`README.md:122-128`): `openclaw-supermemory`, `claude-supermemory`, `opencode-supermemory`, `hermes-agent`.
5. **MCP** — `npx -y install-mcp@latest https://mcp.supermemory.ai/mcp --client claude --oauth=yes` — registers the hosted MCP server. At the original pin it had `memory`, `recall`, `listProjects`, `whoAmI`, `memory-graph`, `fetch-graph-data`. At the current pin the tools are snake_case (`add_memory`, `search_memory`, `get_profile`, `forget_memory`, `get_document`, `list_documents`, `list_memories`, `list_spaces`, `who_am_i`, plus graph, upload and space-selection tools), with camelCase aliases kept for cached clients (`apps/mcp/src/server/tools/index.ts:1-28`), `namespaces`/`profile`/`widget` resources and a `context` prompt.

Telemetry: PostHog is wired through every MCP tool call. At the current pin it uses `@posthog/mcp` instrumentation described in the commit log as metadata-only, grouped by conversation id (`apps/mcp/src/server/analytics.ts:1-7`, #1712, #1718). This is **on by default** for the hosted MCP. The OSS MCP source lets you self-host without it (drop `POSTHOG_API_KEY` from the env).

## Connectors — five providers, the validation enum lists three

The marketing surface (`README.md:52, 294`) lists **six** external connectors: Google Drive, Gmail, Notion, OneDrive, GitHub, Web Crawler. The OAuth state schema (`packages/validation/schemas.ts:126-131`) still enumerates only three: `notion | google-drive | onedrive`. At the original pin Gmail and GitHub appeared in the web app's UI strings and `ImportProvider` type alias (`apps/web/components/settings/sync-utils.ts`, since removed with the web app) but **not in the `ConnectionProviderEnum`** the API validates against. Either (a) the enum is stale, (b) Gmail/GitHub use a different ingest path that bypasses the connection-state machine, or (c) they're not yet wired through this version of the schemas. Treat the README's connector list as marketing-forward.

For the three confirmed connectors, the wire shape is standard OAuth: per-connection access/refresh token, expiry, scoped `containerTags`, `documentLimit` (default 10,000 — `schemas.ts:139, 154`), per-provider metadata. Sync history and trigger types (`event` webhook / `cron` / `manual`) were user-visible in the original-pin dashboard.

The cron trigger on the hosted infrastructure is **every 4 hours for connection imports** (`CLAUDE.md:83`). Webhooks are the real-time path; cron is the fallback.

## Headline benchmark numbers — the #1 claims

Source: `README.md:32, 39, 365-379` at the current pin. All numbers are **self-reported by the maintainer**; I found no independent re-runs in the repo.

| Benchmark | Claim | Detail in repo |
|---|---|---|
| LongMemEval | **#1**; 81.6% at the original pin, now reported as 95% Recall@15 with ~720 tokens of context | `README.md:371, 375` |
| LoCoMo | **#1** (no number given) | `README.md:372` |
| ConvoMem | **#1** | `README.md:373` |
| MemoryBench | Own framework, OSS at supermemoryai/memorybench | `README.md:379-390` |

Three things to surface honestly:

1. **The #1 claims are not in commensurable form.** LongMemEval had a single 81.6% accuracy-style number at the original pin and now has a Recall@15 figure, a different metric. LoCoMo and ConvoMem ship without absolute scores. This makes head-to-head with [[Profile__Mem0]]'s **93.4% LongMemEval / 91.6% LoCoMo** (per Mem0's `README.md:45-68`) and [[Profile__MemPalace]]'s **96.6% R@5 LongMemEval / 92.9% ConvoMem recall** (per MemPalace's `benchmarks/BENCHMARKS.md:1-95`) impossible to do from the public claims alone — different metrics (accuracy vs R@5 vs recall@K), possibly different splits, possibly different judging LLMs.
2. **MemPalace is more transparent about caveats than Supermemory.** The MemPalace BENCHMARKS doc explicitly flags tuning on three wrong answers and reports the held-out generalizable figure separately (`Profile__MemPalace.md` §TL;DR). Supermemory's README lists no methodology caveats. The full eval setup is presumably in the (separate) `supermemoryai/memorybench` repo, not pinned here.
3. **The engine that produces the numbers is closed.** This is the structural concern. A benchmark run requires re-running with a held-out split against your own data, against your own judge. With Mem0, MemPalace, Letta, Neo, Volt — the harness is reproducible because the system is OSS. With Supermemory, you can run their `memorybench` against their hosted API and verify the *score*, but you cannot inspect whether the harness leaked the test set into prompt construction or whether the engine has hot-path code paths conditional on benchmark-shaped inputs. **The right adjudicator for this is [[Profile__StateBench]]** — a system whose numbers stay stable under StateBench's resurrection-rate / scope-leak / must-not-mention dimensions is meaningfully #1; one that doesn't is only #1 on its own card.

Direct comparison with peer headlines, knowing all of them are self-reported:

| System | LongMemEval | LoCoMo | ConvoMem | Methodology readable? |
|---|---|---|---|---|
| Supermemory | 95% R@15 now; 81.6% earlier (#1 claimed) | #1 claimed | #1 claimed | Engine closed |
| [[Profile__Mem0]] (v3) | 93.4% | 91.6% | — | Yes (`evaluation/` dir) |
| [[Profile__MemPalace]] | 96.6% R@5 (98.4% held-out) | — | 92.9% recall | Yes (`benchmarks/`) |

These are **not the same metric** for LongMemEval — Mem0 reports accuracy, MemPalace reports R@5. Three "#1 on LongMemEval" claims can each be true on their own definition. The right reaction is "show me your harness."

## What's inside this submodule

| Path | What's there |
|---|---|
| `apps/web/` | Redirect shell from `app.supermemory.ai` to the console (was the Nova consumer app at the original pin; archived as tag `archive/web-2026-09`) |
| `apps/mcp/` | The `mcp.supermemory.ai` MCP server (Cloudflare Worker + Durable Objects) on the v5 API — snake_case tools, namespace/profile/widget resources, one prompt, an MCP Apps widget, PostHog telemetry, `server.json` for the MCP Registry |
| `apps/sdk-playground/` | Interactive SDK chat playground |
| `apps/memory-graph-playground/` | Standalone demo of the force-directed memory-graph visualizer |
| `apps/docs/` | Mintlify docs site, including the v5 API reference, v3/v4→v5 migration guides, and self-hosting docs for the closed binary |
| `packages/tools/` | `@supermemory/tools` 3.x on v5 — framework adapters: Vercel AI SDK (6 and 7), Mastra, OpenAI Agents, VoltAgent, Claude Memory Tool, shared profile/search client; replaced the retired `@supermemory/ai-sdk` |
| `packages/agent-framework-python/` | `supermemory_agent_framework` — Python framework adapter |
| `packages/openai-sdk-python/` | `supermemory_openai` — OpenAI Agents SDK middleware (Python) |
| `packages/pipecat-sdk-python/` | `supermemory_pipecat` — Pipecat voice-agent middleware |
| `packages/cartesia-sdk-python/` | `supermemory_cartesia` — Cartesia voice middleware |
| `packages/livekit-sdk-python/` | LiveKit Agents memory wrapper (new) |
| `packages/memory-graph/` | Reusable force-directed canvas visualizer (vanilla canvas, no React DOM) consumed by web + MCP UI |
| `packages/validation/` | Zod schemas that mirror the hosted API contract — **the most informative directory in the repo** |
| `packages/lib/` | Shared TS — `better-fetch` client, auth context, types, constants |
| `packages/ui/`, `packages/hooks/` | Design-system primitives consumed by web + MCP UI |
| `skills/supermemory/` | Claude Code skill — `SKILL.md` + five reference markdowns (architecture, api-reference, quickstart, sdk-guide, use-cases) |
| **NOT in the repo** | The memory engine (hosted, or the closed `supermemory-server` binary). The extraction prompt. The hybrid-search ranker. The forgetting policy. The supersession reconciliation. The connector sync workers. |

If you only read one file: `packages/validation/schemas.ts`. It is the closest thing to engine documentation that the OSS repo offers.

## Mental model for using it well

- **Treat it as a closed engine with an unusually good thin-client surface.** The OSS code is for SDK auditing and MCP self-hosting. It is not for understanding (or reproducing) the memory engine.
- **Use the `profile()` call as the recall primitive, not `search()`.** The single-call `{ static, dynamic, searchResults }` shape is the design's distinctive bet and is what the framework adapters route to (`packages/tools/src/shared/memory-client.ts:30-60`). Skipping it for raw search means you're paying for the engine and using it as a vector DB.
- **Always pass a namespace explicitly.** The v5 SDK requires one per call; the MCP server's fallback `sm_project_default` (`apps/mcp/src/server/client/index.ts:22`) is a footgun in any multi-tenant context. Migrate off the v3/v4 `containerTag` calls the README still shows.
- **If you need to verify forgetting, build it into your eval.** Set a `forgetAfter`, query past it, confirm absence. Same with supersession — write v1, write contradictory v2, query, confirm `isLatest` on v2 and v1 unreachable from `searchMode: "memories"`. Do not take the README's "automatic forgetting" claim on faith.
- **The Vercel AI SDK adapter is `addMemory: "always"` by default.** Every turn ships the full conversation to the engine (`packages/tools/src/vercel/middleware.ts:156-180`). For high-volume production this is a real cost line — set `addMemory: "never"` and call `client.add()` only when something memory-worthy happens, or you'll pay for extraction on every "ok" and "thanks."

## When NOT to reach for this

- **You need a self-hosted memory engine whose source you can read.** Supermemory now ships a self-hosted binary that can run offline with a local model, but its source is not public. If you need to read or patch the engine, use [[Profile__Mem0]] (self-hosted server tier), [[Profile__MemPalace]] (local-first, no API key), [[Profile__Volt]] (embedded Postgres), or [[Profile__Letta]] (Docker).
- **You need to audit the extraction policy.** It is closed-source. If your compliance posture requires reading the code that decides what gets stored from a user conversation, this is the wrong system. Use [[Profile__Mem0]] (`mem0/configs/prompts.py:468` is the whole prompt).
- **You don't want LLM-driven extraction at all.** Use [[Profile__MemPalace]] — verbatim chunks, hybrid recall, 0 LLM calls in the write path.
- **You want to read the eviction policy.** Not available. [[Profile__Volt]] and [[Profile__Neo]] are the entries where eviction is inspectable.
- **You need provenance you can verify.** The `MemoryDocumentSource` join table is the right shape, but you can't see the engine code that populates it. [[Profile__Neo]]'s `superseded_by` chain in scoped JSON is verifiable; this is not.
- **Your benchmarking discipline is "reproduce, don't repeat."** Re-running `memorybench` against `api.supermemory.ai` reproduces a score; it does not let you adjudicate the harness. For that, see [[Profile__StateBench]].

## How this compares to the rest of the study

| Axis | Supermemory | [[Profile__Mem0]] | [[Profile__MemPalace]] | [[Profile__StateBench]] |
|---|---|---|---|---|
| **Shape** | Closed engine (hosted or binary) + OSS thin clients | OSS library / server / hosted | OSS library + MCP, local-first | Conformance benchmark |
| **Storage** | Postgres + Cloudflare Workers AI + HNSW (per docs) | Vector + entity-store + SQLite | ChromaDB + SQLite knowledge graph | N/A |
| **Write policy** | Extraction (closed) + version chain in schema | Single-pass extraction (readable) | Append-only verbatim, no extraction | Tests directly |
| **Eviction** | Schema models it (`forgetAfter`, `isForgotten`); policy unverifiable | None in OSS | None by design (infinite retention) | Penalizes wrong eviction |
| **Supersession** | First-class in schema (`parentMemoryId`, `version`, `isLatest`) | Not modeled (hash-dedup only) | Not modeled (additive only) | Measures resurrection rate |
| **Scope model** | namespace (v5; was optional `containerTag`) + Spaces + Org | user/agent/run/actor (required) | wing/room/agent/diary | Tests scope leak |
| **Recall** | Hybrid (RAG+memory) + `profile()` primitive | Hybrid (semantic+BM25+entity boost) | Hybrid (semantic+BM25+closet) | Tests must/must-not |
| **External connectors** | 5 first-class (GDrive/Gmail/Notion/OneDrive/GitHub) | None first-party | None first-party | N/A |
| **Multi-modal** | First-class (PDF/OCR/transcription/AST chunking) | Image/audio via providers | Text + URL primary | N/A |
| **Self-reported #1 on** | LongMemEval / LoCoMo / ConvoMem | LoCoMo 91.6 / LongMemEval 93.4 | LongMemEval 96.6% R@5 / ConvoMem 92.9% | N/A — *is* the yardstick |
| **Engine readable?** | **No** | Yes | Yes | N/A |

## Changes since the previous pin (2026-05-26 → 2026-11-22)

Pin moved from `9229c60a6146` to `fa2af930dd78`: 372 commits. Tags in range are for the self-hosted server binary (`server-v0.0.1`, first tagged 2026-05-30, through `server-v0.0.9`) plus `archive/web-2026-09`; `@supermemory/tools` went from 2.x to 3.2.0. License is still MIT. The verifiability thesis holds: the engine is still not in the repo.

- **Self-hosted engine, closed source.** `supermemory-server` ("Supermemory local") runs the engine as a single binary with an embedded graph store, local embeddings and any OpenAI-compatible model, including fully offline via Ollama. The docs state the binary is built from a non-public codebase and collects telemetry unless disabled (`apps/docs/self-hosting/overview.mdx`, `README.md:314-361`).
- **v5 API.** A breaking, namespace-scoped API: `containerTag` becomes a path namespace (`/ns/{namespace}/document|search|profile`), forgetting and profiles get new shapes, and a `dreaming` ingest mode controls when extraction runs (`apps/docs/migration/api-v5.mdx`, `apps/docs/v5/api-reference/`). The MCP server, `@supermemory/tools` 3.x and the Python wrappers moved to v5 (#1783, #1773, #1786, #1872). The README examples still show the v4 shape.
- **MCP server rewrite.** Code moved from `apps/mcp/src/{server,client}.ts` to `apps/mcp/src/server/`; tools are snake_case with legacy aliases, `get_profile` is a tool, `forget_memory` forgets by ID only instead of by semantic match (#1820), and the active space lives in a Durable Object (`apps/mcp/src/server/tools/index.ts`, `apps/mcp/src/server/client/index.ts:240-345`).
- **Surfaces removed.** The Nova consumer app is now a redirect shell (#1651), the browser extension went with it, the Raycast extension was removed (#1818), and `@supermemory/ai-sdk` was retired (#1773). New: an SDK playground app and a LiveKit Agents Python wrapper.
- **Benchmarks.** The README dropped the 81.6% LongMemEval figure and now leads with "95% Recall@15 with a 99.4% context reduction" alongside the same three #1 claims (`README.md:33, 371-375`); vendor-reported, no harness in this repo.
- **Unchanged.** `packages/validation/schemas.ts` (one line added): the `MemoryEntry` version chain and forget fields, and the three-value `ConnectionProviderEnum`, are where they were.

## One-line summary

> Supermemory ships the most polished thin-client surface in the study (TS + Python SDKs + framework adapters + MCP + Claude Code skill) wrapped around a closed-source engine, hosted or run as a closed binary, that self-reports #1 on the three headline harnesses — making it the operational benchmark for what "memory + RAG as one product" looks like, and the system whose claims you most need a [[Profile__StateBench]]-style independent harness to verify.
