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
date_authored_current_draft: 2026-05-26
lede: >-
  The most polished client surface in the study, wrapped around an engine that
  is not in the repo — every SDK points at a hosted API.
summary: >-
  Profile of Supermemory as pinned in the memory-layers-for-agents study. Its
  role here is as the verifiability case: the entry with the loudest benchmark
  claims and the least inspectable engine. Covers what the OSS repo actually
  ships (Next.js consumer app, Cloudflare Durable Object MCP server, seven SDKs
  and framework adapters, the Zod validation schemas, a memory-graph
  visualizer), the MemoryEntry schema with its version chain and forget
  metadata, the two ingest endpoints, the profile() call as the distinctive
  recall primitive, containerTag scoping and its unsafe default, the connector
  discrepancy between the README and the validated provider enum, and an honest
  reading of three mutually incommensurable "#1" claims. The recommended posture
  is in the mental-model section: verify forgetting and supersession in your own
  eval rather than on the README's word.
publish: true
---

# Supermemory — Profile

A profile of Supermemory as it lives in this study (`studies/memory-layers-for-agents/supermemory/`). Cites pinned paths so you can jump to source rather than trust paraphrase. Read alongside [`Profile__Mem0.md`](./Profile__Mem0.md) — Supermemory is the closest peer in scope and is the entry making the strongest benchmark claims in the field — and [`Profile__MemPalace.md`](./Profile__MemPalace.md), which makes a structurally opposite extraction-vs-verbatim bet on the same benchmark.

## TL;DR

Supermemory positions itself as the **state-of-the-art memory and context engine for AI**, and pins the claim on three numbers (`README.md:28`, `README.md:306-308`): **#1 on LongMemEval (81.6%), #1 on LoCoMo, #1 on ConvoMem** — the three headline harnesses the rest of the study competes on. The repo is the most active OSS entry in the study after Mem0 and Letta (**1,637 commits**, ~22.7k stars at time of profile), MIT-licensed.

The bet is **universal RAG + memory in one engine**: fact extraction from conversations, auto-maintained user profiles (static + dynamic), hybrid semantic search that interleaves user memories with knowledge-base chunks, multi-modal extractors (PDF/OCR/transcription/AST-aware code chunking), and real external connectors that webhook-sync Google Drive / Gmail / Notion / OneDrive / GitHub into the same store (`README.md:36-42`, `README.md:280`).

The most consequential thing to know reading the repo: **the memory engine is not in it.** Every SDK and surface in `supermemory/` points at `https://api.supermemory.ai` (`apps/mcp/src/server.ts:528`, `apps/mcp/src/client.ts:113`, `packages/lib/api.ts:320`, `packages/tools/src/conversations-client.ts:76`, `packages/tools/src/shared/memory-client.ts:41`). The extraction prompt, the supersession reconciliation logic, the hybrid-search ranker, the eviction policy — all closed-source. What is in the repo is (a) the Next.js consumer app (Nova), (b) the MCP server (a Cloudflare Durable Object that proxies to the hosted API), (c) seven SDKs (TS + four Python language-specific wrappers + framework adapters for Vercel AI SDK / Mastra / OpenAI Agents / VoltAgent), (d) the Zod schemas the hosted API validates against, (e) a Claude Code skill, and (f) a force-directed memory-graph visualizer.

**One sentence:** *Supermemory is a closed-source hosted memory engine whose OSS repo ships an unusually complete thin-client surface (TS + Python SDKs, framework adapters, MCP, consumer app, browser extension, Claude Code skill), making it the most operationally polished entry in the study — and the hardest to verify, because the engine that produces the benchmark numbers is not in the codebase you can read.*

## Why this exists — the design bet

The thesis is laid out plainly in `README.md:34-44` and elaborated in `skills/supermemory/references/architecture.md:5-36`: agents should not be choosing between "RAG" (stateless document retrieval) and "memory" (per-user state). The bet is that there is one *living knowledge graph* — extracted facts, ingested documents, connector-synced content — that the agent queries with one API, getting both knowledge-base hits and personalized context in the same response (`README.md:343`).

Five design moves follow:

1. **Memory is extracted, not stored verbatim.** Conversations go through the engine and come back as discrete `MemoryEntry` rows with parent/version chains. This is the direct opposition to [[Profile__MemPalace]] (verbatim chunks, no extraction) and aligned with [[Profile__Mem0]] (single-pass extraction).
2. **User profile is the recall primitive, not just memories.** The headline `client.profile()` call returns `{ profile.static, profile.dynamic, searchResults }` in one round-trip, ~50ms (`README.md:200-208`, `apps/mcp/src/server.ts:619-697`). Most peers force you to compose this from `search` calls; Supermemory bakes it in.
3. **Hybrid search interleaves RAG and memory by default.** `searchMode: "hybrid"` is the default; `"memories"` is opt-in (`README.md:248`, `apps/mcp/src/client.ts:227`). The marketing pitch — "Memory is not RAG" (`README.md:343`) — is contradicted by the architectural pitch, which is "RAG and memory together."
4. **Supersession and forgetting are first-class in the schema.** `MemoryEntry` carries `version`, `parentMemoryId`, `rootMemoryId`, `isLatest`, `isForgotten`, `forgetAfter`, `forgetReason` (`packages/validation/schemas.ts:242-278`). Whether the engine *uses* these fields well is unverifiable from the OSS code — only the schema is.
5. **Connectors are part of the engine, not a separate product.** Real-time webhook sync from five external SaaS providers feeds the same store as the chat memories (`README.md:280`, `apps/web/components/settings/sync-utils.ts:38-54`).

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
| Embedding model | Cloudflare Workers AI (per `CLAUDE.md:80`) | "Vector embedding generation using Cloudflare AI" | `CLAUDE.md:73-82` |
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

1. **`POST /v3/documents`** — unstructured content (URL, PDF, image, video, text). Triggers the six-stage `IngestContentWorkflow` (queued → extracting → chunking → embedding → indexing → done) per `CLAUDE.md:70-75` and `skills/supermemory/references/architecture.md:37-168`. Documents are the unit of upload; memories are the *output* of extraction.
2. **`POST /v4/conversations`** — structured chat (`packages/tools/src/conversations-client.ts:73-102`). Takes `{ conversationId, messages, containerTags, metadata, entityContext }`. The endpoint comment claims it "supports structured messages with smart diffing and append detection on the backend" (`conversations-client.ts:7`) — i.e., the engine handles incremental updates of the same conversation. Cannot be verified from this repo.

The framework adapters write via `/v4/conversations`. The Vercel AI SDK wrapper (`packages/tools/src/vercel/middleware.ts:78-113`) saves the conversation **after** every assistant response in a fire-and-forget call, with `addMemory: "always"` as the default (`packages/tools/src/vercel/index.ts:135`). This is closer to [[Profile__Letta]]'s self-editing pattern than to [[Profile__Mem0]]'s "extract per turn" — Supermemory ships the full conversation each round and lets the engine decide what's new.

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

Source: `apps/mcp/src/client.ts:217-256` (memory search), `apps/mcp/src/client.ts:259-299` (profile), `packages/tools/src/shared/memory-client.ts:24-65` (the `/v4/profile` POST). The MCP recall handler returns memories with a similarity percentage and a `(N% match)` annotation per result (`apps/mcp/src/server.ts:660-664`).

Filter operators on `search.documents` mirror [[Profile__Mem0]]'s richer set: `AND`/`OR` boolean groups with `numericOperator` (`>`/`>=`/`<`/`<=`/`=`) over string/number/boolean values (`packages/validation/api.ts:251-323`). This is the metadata filter shape the engine accepts on the wire.

The retrieval mechanism in the skill docs (`skills/supermemory/references/architecture.md:204-244`) describes the search path as **embed → vector lookup → threshold filter → relationship expansion (follow `extends`/`derives`/`updates` from candidate memories) → rank** by similarity + recency + static-priority + relationship-strength. Again — the doc is marketing, not source. The actual implementation is closed.

## Eviction & compaction — schema says yes, implementation unverifiable

The `MemoryEntry` schema models forgetting end-to-end (`schemas.ts:262-266`): `isForgotten`, `forgetAfter` (TTL date), `forgetReason` (free-text). The MCP server exposes a `memory(action: "forget")` tool that calls `client.memories.forget({ content | id, containerTag })` with an exact-match-then-semantic-fallback pattern at similarity ≥ 0.85 (`apps/mcp/src/client.ts:147-214`). The marketing claim is bolder (`README.md:38`):

> Extracts facts from conversations. Handles **temporal changes, contradictions, and automatic forgetting**.

…and (`README.md:345`):

> Automatic forgetting. Supermemory knows when memories become irrelevant. Temporary facts ("I have an exam tomorrow") expire after the date passes. Contradictions are resolved automatically. Noise never becomes permanent memory.

**Verdict on whether this is real or aspirational:** the *schema* is real (the fields exist; the documents API returns them; the visualizer renders chains). The *policy* — when the engine decides to set `forgetAfter`, when contradictions trigger a new version vs. an explicit forget, what counts as "noise" — is invisible. Compare to [[Profile__Mem0]] (no eviction at all in OSS), [[Profile__Volt]] (deterministic three-level escalation that you can read), [[Profile__Neo]] (`prune_stale_facts` / `demote_unhelpful_facts` as readable Python). Supermemory's forgetting is the only one in the study where the *user can't verify it works*. That is structurally a research integrity concern for any benchmark that claims it.

## Scopes & namespacing

The unit is **`containerTag`** (`apps/mcp/src/server.ts:64-72`, `packages/validation/api.ts:122-130`) — a free-form string, max 128 chars. By convention:

- A user ID is the typical "root" container (the MCP OAuth flow attaches one).
- Additional tags scope memories to projects (the MCP `listProjects` tool exposes them, cached 5 min — `apps/mcp/src/server.ts:27, 772-790`).
- A memory can carry multiple `containerTags` (`MemorySchema.containerTags: z.array(z.string())`) — i.e., scopes overlap by design.

Layered above container tags are **Spaces** (`schemas.ts:218-237`) with a separate access-control table (`SpacesToMembersSchema:299-307`) carrying owner/admin/editor/viewer roles. Spaces are the multi-user share primitive; container tags are the single-user partition.

Above Spaces is **Organization** (Better Auth-managed per `CLAUDE.md:53-54`). The `MemoryEntry` carries `orgId` + `userId` at the row level; tenancy is enforced at the engine, not the client.

This is more elaborate than [[Profile__Mem0]] (user/agent/run/actor — required at every call) and more elaborate than [[Profile__MemPalace]] (wing/room/agent/diary). The trade-off: Supermemory's scope is *optional* at the API surface (defaulted to `sm_project_default` when omitted — `apps/mcp/src/client.ts:4, 121`), so the same "I forgot to scope and leaked Alice's memories to Bob" failure mode Mem0 designs against is **possible here**.

## Operational story

Three tiers, but in a different shape from Mem0's library/server/hosted split — because **the library and server are not in the OSS repo**.

1. **Hosted API** — the only memory engine. `https://api.supermemory.ai`. v3 for documents, v4 for conversations + profile + search. Authentication via Bearer token from `console.supermemory.ai`. Free tier exists per the 402 handler in `apps/mcp/src/client.ts:388`. This is the *only* deployment for actual memory.
2. **Self-hosted thin clients** — what you can run from this repo:
   - `apps/web/` — Next.js consumer app (Nova), `bun run dev`, deploys to Cloudflare via OpenNext (`apps/web/package.json:8-15`).
   - `apps/mcp/` — Cloudflare Durable Object MCP server, the `https://mcp.supermemory.ai/mcp` endpoint open-sourced for review (`README.md:143`).
   - `apps/browser-extension/` — WXT-based, twitter-utils included (`apps/browser-extension/utils/twitter-utils.ts`).
   - `apps/raycast-extension/` — Raycast quick-launch.
   - `apps/memory-graph-playground/` — visualizer demo.
3. **SDKs** (npm + PyPI):
   - **TS:** `supermemory` (closed-source SDK on npm; this repo *consumes* it), `@supermemory/tools` (the framework adapters in this repo).
   - **Python:** `supermemory` (closed-source SDK on PyPI), `supermemory_agent_framework`, `supermemory_openai`, `supermemory_pipecat`, `supermemory_cartesia` — each a separate framework adapter package shipped from `packages/*-python/`.
4. **Editor plugins** — separate repos referenced from the README (`README.md:106-109`): `openclaw-supermemory`, `claude-supermemory`, `opencode-supermemory`, `hermes-agent`.
5. **MCP** — `npx -y install-mcp@latest https://mcp.supermemory.ai/mcp --client claude --oauth=yes` (`README.md:114`) — registers the hosted MCP server with eight tools (`memory`, `recall`, `listProjects`, `whoAmI`, `memory-graph`, `fetch-graph-data`, plus `User Profile` + `My Projects` resources and a `context` prompt — `apps/mcp/src/server.ts:106-514`).

Telemetry: PostHog is wired through every MCP tool call with `posthog.memoryAdded` / `memoryForgot` / `memorySearch` events carrying `userId`, `mcp_client_name`, `mcp_client_version`, `containerTag`, `content_length`, `search_duration_ms` (`apps/mcp/src/server.ts:548-684`). This is **on by default** for the hosted MCP — anyone running through `mcp.supermemory.ai` is being measured. The OSS MCP source lets you self-host without it (drop `POSTHOG_API_KEY` from the env).

## Connectors — five providers, the validation enum lists three

The marketing surface (`README.md:41, 280`, `apps/web/components/settings/sync-utils.ts:38-54`) lists **six** external connectors: Google Drive, Gmail, Notion, OneDrive, GitHub, Web Crawler — plus S3 as a private extension. The OAuth state schema (`packages/validation/schemas.ts:126-131`) authoritatively enumerates only three: `notion | google-drive | onedrive`. Gmail and GitHub appear in the UI strings and the `ImportProvider` type alias (`sync-utils.ts:48-54`) but **do not appear in the `ConnectionProviderEnum`** the API validates against. Either (a) the enum is stale, (b) Gmail/GitHub use a different ingest path that bypasses the connection-state machine, or (c) they're not yet wired through this version of the schemas. Treat the README's connector list as marketing-forward.

For the three confirmed connectors, the wire shape is standard OAuth: per-connection access/refresh token, expiry, scoped `containerTags`, `documentLimit` (default 10,000 — `schemas.ts:154`), per-provider metadata. Sync history and trigger types (`event` webhook / `cron` / `manual` — `sync-utils.ts:32-36`) are user-visible in the dashboard.

The cron trigger on the hosted infrastructure is **every 4 hours for connection imports** (`CLAUDE.md:82`). Webhooks are the real-time path; cron is the fallback.

## Headline benchmark numbers — the #1 claims

Source: `README.md:28, 300-308`. All numbers are **self-reported by the maintainer**; I found no independent re-runs in the repo.

| Benchmark | Claim | Detail in repo |
|---|---|---|
| LongMemEval | **81.6% — #1** | `README.md:306` |
| LoCoMo | **#1** (no number given) | `README.md:307` |
| ConvoMem | **#1** | `README.md:308` |
| MemoryBench | Own framework, OSS at supermemoryai/memorybench | `README.md:310-314` |

Three things to surface honestly:

1. **The #1 claims are not in commensurable form.** LongMemEval has a single 81.6% number. LoCoMo and ConvoMem ship without absolute scores. This makes head-to-head with [[Profile__Mem0]]'s **93.4% LongMemEval / 91.6% LoCoMo** (per Mem0's `README.md:45-68`) and [[Profile__MemPalace]]'s **96.6% R@5 LongMemEval / 92.9% ConvoMem recall** (per MemPalace's `benchmarks/BENCHMARKS.md:1-95`) impossible to do from the public claims alone — different metrics (accuracy vs R@5 vs recall@K), possibly different splits, possibly different judging LLMs.
2. **MemPalace is more transparent about caveats than Supermemory.** The MemPalace BENCHMARKS doc explicitly flags tuning on three wrong answers and reports the held-out generalizable figure separately (`Profile__MemPalace.md` §TL;DR). Supermemory's README lists no methodology caveats. The full eval setup is presumably in the (separate) `supermemoryai/memorybench` repo, not pinned here.
3. **The engine that produces the numbers is closed.** This is the structural concern. A benchmark run requires re-running with a held-out split against your own data, against your own judge. With Mem0, MemPalace, Letta, Neo, Volt — the harness is reproducible because the system is OSS. With Supermemory, you can run their `memorybench` against their hosted API and verify the *score*, but you cannot inspect whether the harness leaked the test set into prompt construction or whether the engine has hot-path code paths conditional on benchmark-shaped inputs. **The right adjudicator for this is [[Profile__StateBench]]** — a system whose numbers stay stable under StateBench's resurrection-rate / scope-leak / must-not-mention dimensions is meaningfully #1; one that doesn't is only #1 on its own card.

Direct comparison with peer headlines, knowing all of them are self-reported:

| System | LongMemEval | LoCoMo | ConvoMem | Methodology readable? |
|---|---|---|---|---|
| Supermemory | 81.6% (#1 claimed) | #1 claimed | #1 claimed | Engine closed |
| [[Profile__Mem0]] (v3) | 93.4% | 91.6% | — | Yes (`evaluation/` dir) |
| [[Profile__MemPalace]] | 96.6% R@5 (98.4% held-out) | — | 92.9% recall | Yes (`benchmarks/`) |

These are **not the same metric** for LongMemEval — Mem0 reports accuracy, MemPalace reports R@5. Three "#1 on LongMemEval" claims can each be true on their own definition. The right reaction is "show me your harness."

## What's inside this submodule

| Path | What's there |
|---|---|
| `apps/web/` | Next.js consumer app (Nova) — chat UI, onboarding, settings, integrations, MCP install flow, dashboard analytics, OpenNext for Cloudflare deploy |
| `apps/mcp/` | The `mcp.supermemory.ai` Cloudflare Durable Object MCP server — eight tools, three resources, one prompt, PostHog telemetry |
| `apps/browser-extension/` | WXT-based browser extension with Twitter ingest utilities |
| `apps/raycast-extension/` | macOS Raycast quick-launch |
| `apps/memory-graph-playground/` | Standalone demo of the force-directed memory-graph visualizer |
| `apps/docs/` | Mintlify docs site |
| `packages/tools/` | `@supermemory/tools` — framework adapters: Vercel AI SDK, Mastra, OpenAI Agents, VoltAgent, Claude Memory Tool, shared profile/search client |
| `packages/ai-sdk/` | `@supermemory/ai-sdk` — Vercel AI SDK tool definitions |
| `packages/agent-framework-python/` | `supermemory_agent_framework` — Python framework adapter |
| `packages/openai-sdk-python/` | `supermemory_openai` — OpenAI Agents SDK middleware (Python) |
| `packages/pipecat-sdk-python/` | `supermemory_pipecat` — Pipecat voice-agent middleware |
| `packages/cartesia-sdk-python/` | `supermemory_cartesia` — Cartesia voice middleware |
| `packages/memory-graph/` | Reusable force-directed canvas visualizer (vanilla canvas, no React DOM) consumed by web + MCP UI |
| `packages/validation/` | Zod schemas that mirror the hosted API contract — **the most informative directory in the repo** |
| `packages/lib/` | Shared TS — `better-fetch` client, auth context, types, constants |
| `packages/ui/`, `packages/hooks/` | Design-system primitives consumed by web + MCP UI |
| `skills/supermemory/` | Claude Code skill — `SKILL.md` + five reference markdowns (architecture, api-reference, quickstart, sdk-guide, use-cases) |
| **NOT in the repo** | The memory engine. The extraction prompt. The hybrid-search ranker. The forgetting policy. The supersession reconciliation. The connector sync workers. |

If you only read one file: `packages/validation/schemas.ts`. It is the closest thing to engine documentation that the OSS repo offers.

## Mental model for using it well

- **Treat it as a hosted API with an unusually good thin-client surface.** The OSS code is for SDK auditing, MCP self-hosting, and consumer-app customization. It is not for understanding (or reproducing) the memory engine.
- **Use the `profile()` call as the recall primitive, not `search()`.** The single-call `{ static, dynamic, searchResults }` shape is the design's distinctive bet and is what the framework adapters route to (`packages/tools/src/shared/memory-client.ts:41`). Skipping it for raw search means you're paying for the engine and using it as a vector DB.
- **Always pass a `containerTag` explicitly.** The default `sm_project_default` (`apps/mcp/src/client.ts:4`) is a footgun in any multi-tenant context. The schema makes scope optional; you have to enforce it yourself.
- **If you need to verify forgetting, build it into your eval.** Set a `forgetAfter`, query past it, confirm absence. Same with supersession — write v1, write contradictory v2, query, confirm `isLatest` on v2 and v1 unreachable from `searchMode: "memories"`. Do not take the README's "automatic forgetting" claim on faith.
- **The Vercel AI SDK adapter is `addMemory: "always"` by default.** Every turn ships the full conversation to `/v4/conversations` (`packages/tools/src/vercel/middleware.ts:78-113`). For high-volume production this is a real cost line — set `addMemory: "never"` and call `client.add()` only when something memory-worthy happens, or you'll pay for extraction on every "ok" and "thanks."

## When NOT to reach for this

- **You need an air-gapped or self-hosted memory engine.** Supermemory does not ship one. Use [[Profile__Mem0]] (self-hosted server tier), [[Profile__MemPalace]] (local-first, no API key), [[Profile__Volt]] (embedded Postgres), or [[Profile__Letta]] (Docker).
- **You need to audit the extraction policy.** It is closed-source. If your compliance posture requires reading the code that decides what gets stored from a user conversation, this is the wrong system. Use [[Profile__Mem0]] (`mem0/configs/prompts.py:468` is the whole prompt).
- **You don't want LLM-driven extraction at all.** Use [[Profile__MemPalace]] — verbatim chunks, hybrid recall, 0 LLM calls in the write path.
- **You want to read the eviction policy.** Not available. [[Profile__Volt]] and [[Profile__Neo]] are the entries where eviction is inspectable.
- **You need provenance you can verify.** The `MemoryDocumentSource` join table is the right shape, but you can't see the engine code that populates it. [[Profile__Neo]]'s `superseded_by` chain in scoped JSON is verifiable; this is not.
- **Your benchmarking discipline is "reproduce, don't repeat."** Re-running `memorybench` against `api.supermemory.ai` reproduces a score; it does not let you adjudicate the harness. For that, see [[Profile__StateBench]].

## How this compares to the rest of the study

| Axis | Supermemory | [[Profile__Mem0]] | [[Profile__MemPalace]] | [[Profile__StateBench]] |
|---|---|---|---|---|
| **Shape** | Closed engine + OSS thin clients | OSS library / server / hosted | OSS library + MCP, local-first | Conformance benchmark |
| **Storage** | Postgres + Cloudflare Workers AI + HNSW (per docs) | Vector + entity-store + SQLite | ChromaDB + SQLite knowledge graph | N/A |
| **Write policy** | Extraction (closed) + version chain in schema | Single-pass extraction (readable) | Append-only verbatim, no extraction | Tests directly |
| **Eviction** | Schema models it (`forgetAfter`, `isForgotten`); policy unverifiable | None in OSS | None by design (infinite retention) | Penalizes wrong eviction |
| **Supersession** | First-class in schema (`parentMemoryId`, `version`, `isLatest`) | Not modeled (hash-dedup only) | Not modeled (additive only) | Measures resurrection rate |
| **Scope model** | `containerTag` (optional, free-form) + Spaces + Org | user/agent/run/actor (required) | wing/room/agent/diary | Tests scope leak |
| **Recall** | Hybrid (RAG+memory) + `profile()` primitive | Hybrid (semantic+BM25+entity boost) | Hybrid (semantic+BM25+closet) | Tests must/must-not |
| **External connectors** | 5 first-class (GDrive/Gmail/Notion/OneDrive/GitHub) | None first-party | None first-party | N/A |
| **Multi-modal** | First-class (PDF/OCR/transcription/AST chunking) | Image/audio via providers | Text + URL primary | N/A |
| **Self-reported #1 on** | LongMemEval / LoCoMo / ConvoMem | LoCoMo 91.6 / LongMemEval 93.4 | LongMemEval 96.6% R@5 / ConvoMem 92.9% | N/A — *is* the yardstick |
| **Engine readable?** | **No** | Yes | Yes | N/A |

## One-line summary

> Supermemory ships the most polished thin-client surface in the study (TS + four Python SDKs + framework adapters + MCP + consumer app + browser extension + Claude Code skill) wrapped around a closed-source hosted engine that self-reports #1 on the three headline harnesses — making it the operational benchmark for what "memory + RAG as one product" looks like, and the system whose claims you most need a [[Profile__StateBench]]-style independent harness to verify.
