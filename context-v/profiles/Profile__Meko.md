---
name: Meko Profile
slug: meko
upstream: https://github.com/yugabyte/meko
package: meko (Go binary, installer at mekodata.ai/install.sh; release meko-v1.20261123.0)
license: Apache-2.0
maintainer: Yugabyte (`yugabyte` org)
study: studies/memory-layers-for-agents
profile_path: studies/memory-layers-for-agents/meko
profile_kind: single-binary server + CLI + MCP/REST + context-engine child process + coding-agent extension (Kode)
date_created: 2026-11-23
site_uuid: 3cf2c49d-e715-4486-a175-fa91a9b558c3
hex_code: i641xr
date_authored_initial_draft: 2026-11-23
date_authored_current_draft: 2026-11-23
lede: >-
  A database company's memory layer that runs locally on SQLite, not YugabyteDB:
  mem0 in a Node child process with its LLM ripped out, governed datapacks as
  the unit of scope, and the user's own Claude Code doing the extraction.
summary: >-
  Profile of Meko (Yugabyte) as pinned in the memory-layers-for-agents study at
  7bb83da (2026-11-23). Meko bundles memories, knowledge files, conversation
  turns, decision traces and artifacts into named datapacks, served by one Go
  binary over MCP and REST with Meko Cloud's wire shapes. Covers the local
  storage engine (SQLite with sqlite-vec and FTS5, partitioned by datapack, a
  200 MB hard cap), the memory schema and its learning-to-collective-memory
  promotion, the Node context engine that wraps upstream mem0 with a "none"
  LLM provider, headless-Claude extraction from the user's messages only,
  Context Gaps, the per-project dedicated-plus-shared datapack model in the
  Kode extension, MEKO.md as a search-before-work index, and the fact that
  sharing roles and the gotchas miner live only in the private Meko Cloud.
publish: true
title: Meko — Profile
date_modified: 2026-11-23
authors:
  - Michael Staton
status: Reference
type: Profiles
---

# Meko — Profile

A profile of Meko as it lives in this study (`studies/memory-layers-for-agents/meko/`), pinned at `7bb83da` (2026-11-23, one commit past the `meko-v1.20261123.0` tag). Every path below is relative to the submodule. Read alongside [[Profile__Mem0]], because Meko's context engine *is* upstream mem0 with its LLM removed, and [[Profile__Paxm]] for the other entry that rides on top of the user's coding-agent harness rather than shipping its own model.

## TL;DR

Meko calls itself "The Context Engine for Agentic Applications": "collective memory, shared knowledge, and decision traces in governed datapacks", run locally as "a single binary that has the command line, the server, the MCP tools and the REST API" (`README.md:17-21`). It comes from Yugabyte, but the local product does **not** run on YugabyteDB. It owns "a SQLite store with vector and full-text search" (`src/meko/docs/design.md:3-5`): mattn/go-sqlite3 from a `yugabyte/go-sqlite3` fork with sqlite-vec compiled in (`design.md:174-176, 196-208`), every searchable table partitioned by `datapack_id` (`design.md:286-288`), and a hard 200 MB content cap past which writes are refused and "Meko Cloud is for larger datasets" (`design.md:16-20, 292-306`).

The retrieval code is mem0. The Node context engine is "mem0's TypeScript library in a Node child process", pinned to `mem0ai` 3.3.1, that "embeds with a local model and makes no LLM calls" (`src/meko/docs/context-engine.md:11-13, 28`). It registers a `none` LLM provider that throws (`context-engine.md:34`) and stores everything back through a Go store API on a unix socket (`design.md:367-376`). Extraction happens outside mem0, by shelling out to the user's own `claude -p` (`src/meko/docs/memory-extraction.md:3-5`).

The idea worth taking is the **datapack**: a named, UUID-keyed workspace that holds memories, knowledge, conversations and artifacts together (`README.md:123-128`), with a `MEKO.md` file in each repo folder listing which datapacks to search before touching the code (`MEKO.md:3-5`).

## Why this exists: the design bet

The bet is that **agent context is a governed, shareable asset that moves between a laptop and a hosted service without changing shape.** The local backend copies Meko Cloud's wire contract: "the `{data, error}` envelope, MCP tool names and replies, REST paths", so "a datapack can move to Cloud without losing anything", while "Meko Cloud's code is private: nothing here copies it or cites its paths" (`design.md:22-25`). Ids are Cloud's UUIDs for the same reason (`design.md:263-264`). With Cloud linked, `/mcp` serves Cloud's tool list and forwards every call that doesn't name a local datapack (`src/meko/docs/cloud.md:115-135`). Moving a local datapack to Cloud is a one-way, all-or-nothing sync that deletes the local copy (`cloud.md:312-327`).

So the open-source repo is the **local edge** of a commercial product. The 200 MB cap, single-owner local datapacks, and Cloud-only sharing all push teams toward Cloud. Read the local engine as a faithful but deliberately small reimplementation.

***

## Storage topology: one SQLite file, partitioned by datapack

- **One database**, `~/.meko/meko.db` (WAL, 0600), plus knowledge-file bytes under `knowledgebase-files/<datapack>/<sha256>` (`design.md:118-139`). One write connection and a read pool of 8, with at most 4 concurrent search scans (`design.md:229-233`).
- **Three indexes on one table set.** `vec0` virtual tables (`memories_vec`, `entities_vec`, `message_chunks_vec`, `kb_chunks_vec`) declare `datapack_id TEXT partition key` and cosine distance (`context-engine.md:119-124`). Contentless FTS5 carries `datapack_id` as a column "so the filter runs inside MATCH (a join with the filter outside FTS never finished at 200 MB)" (`design.md:286-288`; `src/meko/internal/store/migrations/0002_memory.sql:30-36`). An `entities` table backs mem0's entity boost (`0002_memory.sql:52-65`).
- **Measured.** At 200 MB in one datapack, memory search is 771 ms p50, 419 ms of it a brute-force vector scan; sqlite-vec 0.1.9 "scans every vector of the partition, at about 1.1 ms per 1,000" (`context-engine.md:191-209`). That scan cost is the stated reason for the cap: one limit for the whole install, because "each FTS5 term's list holds every datapack's rows" (`design.md:299`).
- **Models are local and locked.** `Xenova/all-MiniLM-L6-v2` (384 dims, q8) on transformers.js, "locked at first start" because re-embedding isn't built (`context-engine.md:22, 33`). Knowledge search adds a local `ms-marco-MiniLM-L-6-v2` cross-encoder over the top 20 RRF candidates (`src/meko/docs/knowledge.md:33, 117`).

## Schema of a memory

```sql
CREATE TABLE memories (
  rowid INTEGER PRIMARY KEY,  id TEXT NOT NULL UNIQUE,   -- UUID, made by mem0
  datapack_id TEXT NOT NULL REFERENCES datapacks(id) ON DELETE CASCADE,
  user_id, agent_id, conversation_id, role, actor_id,     -- '' when absent
  text TEXT NOT NULL, text_lemmatized, hash,
  payload TEXT NOT NULL DEFAULT '{}',                    -- rest of mem0's payload
  source TEXT NOT NULL DEFAULT 'add',                    -- 'add' | 'extract'
  created_at, updated_at                                 -- RFC 3339 UTC
) STRICT;
```
(`src/meko/internal/store/migrations/0002_memory.sql:8-26`, abridged.) `0005_learnings_artifacts.sql:6, 9` adds `promotion_action` (`''` for a learning, `keep_as_memory` once kept) and `connector_id`, a provenance column that "nothing sets ... until connectors exist" (`src/meko/docs/api.md:527-529`). mem0's change log lands in `memory_history(previous_value, new_value, action ADD|UPDATE|DELETE)` (`0002_memory.sql:69-79`).

A memory has **three lifecycle states**, and they are the most distinctive part of the schema. Every memory starts as a *learning*; a person can *keep* it; promotion copies it into `promoted_memories`, deletes it from `memories` through the context engine, and inserts it as a knowledge-base chunk with its own vector, so it becomes the datapack's **collective memory** (`api.md:504-526`). A separate table "hides promoted memories from search without a filter on every scan" (`api.md:513-514`).

No confidence score, no validity interval, no supersession pointer. Compare RetainDB's `validFrom/validUntil` or Neo's supersession edges.

## Write policy

- **Verbatim by default.** `memory_add` stores the text as given and links no entities (`context-engine.md:6, 35`). Update and delete are "the writer's agent only" (`api.md:291`).
- **Extraction is add-only, from the user's side only.** A queue fills as turns arrive; every 5 minutes a run sends up to 30 turns per conversation, plus the 20 nearest existing memories and 4 turns of context, to headless Claude Code with a JSON schema (`memory-extraction.md:9-20`). Claude runs with no tools, no MCP, no saved session, in an empty temp folder (`src/meko/internal/memory/extractor.go:129-150`), on Haiku by default (`extractor.go:33`). The prompt says "Extract from the USER's messages in the new turns only ... Only add: never restate, update or contradict an existing memory here" (`src/meko/internal/memory/extractprompt.go:14-30`). A datapack can pick a `coding` profile and add its own instructions (`extractprompt.go:33-45`; `memory-extraction.md:46`).
- **Dedup is a cosine cut.** A fact with cosine similarity of 0.9 or more to any of its 5 nearest memories is dropped (`extractor.go:38-39, 592-600`). Nothing reconciles contradictions.
- **The user's own LLM, on purpose.** "Meko local runs beside a strong LLM already: the user's agent. No model to download, no key to keep ... A small local model extracts poorly from coding conversations" (`memory-extraction.md:44`).

## Scopes and namespacing

The datapack is the hard boundary: every table carries `datapack_id`, every vector partition and FTS query is keyed on it, and deleting a datapack cascades (`0002_memory.sql:11`; `context-engine.md:119-124`). Inside a datapack, `user_id`, `agent_id` and `conversation_id` are columns, but they don't gate reads the way the datapack does: `memory_search` returns "every agent's memories" in the datapack (`api.md:288`), while writes and conversations belong to their agent (`api.md:291, 296`; `context-engine.md:36`). Above datapacks sit **projects** and **folders** (project, folder, datapack, no deeper), which are filing only: "deleting a project or a folder deletes structure, never content" (`api.md:465-485`).

ACLs are where local and Cloud split. Locally every datapack row reports `grant: "owner"` (`api.md:299`), every project is the local user's (`api.md:495-496`), and every MCP/REST client shares one client token ("Per-client API keys aren't built yet", `api.md:74`). Cloud adds `owner`, `maintainer`, `contributor` and `viewer` grants (`cloud.md:289-290`). Kode's sharing doc gives the semantics: contributors "write their own memories", maintainers "can also promote memories and invite others", viewers are read-only, and "a member sees their own memories plus the datapack's shared knowledge (promoted memories and knowledge-base files), not the owner's raw memories" (`src/extensions/agentk/docs/design/sharing.md:5-10`). Sharing a local datapack returns 409 `DATAPACK_LOCAL` (`sharing.md:14-21`).

## Recall surface

- **MCP at `/mcp`**, stateless streamable HTTP on the official Go SDK (`api.md:230-242`). Tools include `context_search`, `memory_add/search/get_all/get_by_id/update/promote/delete_*`, `conversation_*`, `datapack_*`, `project_*`, `artifact_put/get` and `knowledgebase_*` (`api.md:284-320`).
- **`context_search` is the main call.** It runs memory, knowledge-base and conversation search concurrently and returns `{results: {knowledge_base, memory, conversation: {hits, thread}}, token_savings}` (`api.md:286, 322-337`).
- **Memory score.** `min((semantic + bm25 + boost) / (1 + [bm25>0] + 0.5·[boost>0]), 1)`, with each memory divided only by what it earned (`src/meko/context_engine/src/scoring.ts:1-21`). Search asks mem0 for `max(limit×16, 240)` candidates and rescores them all (`context-engine.md:157-161`).
- **Conversation search** chunks turns into 1024-character windows, keeps hits within cosine distance 0.6 from other conversations, and returns a thread of the top 3 conversations' recent turns (`context-engine.md:37`).
- **Context Gaps.** Without any model judging the reply, a search that returns nothing, or only memories scoring below 0.75, is logged as a question, grouped by normalized query (`api.md:151-175`). A person or an agent can then *teach* an answer, saved as a memory tagged `kind: "taught"` (`api.md:198-205`). This is the only entry in the study that records recall misses as a work queue.

## Eviction and compaction

None by policy. The 200 MB cap refuses writes and "nothing is deleted to get under it" (`design.md:305`). The call log is pruned at 90 days and call contents at 7 (`api.md:101-103, 122-123`), and Context Gaps asks at 90 (`api.md:186`). Memories live until deleted or promoted.

## Serialization on disk

Truth is SQLite with `STRICT` tables and append-only numbered migrations, backed up with `VACUUM INTO` before each upgrade (`design.md:239-245`). Export is one zip per datapack: `datapack.json` plus `memories.jsonl`, `collective_memories.jsonl`, `conversations.jsonl`, `knowledge.jsonl`, `artifacts.jsonl` and `call_log.jsonl`, with embeddings and file bytes included (`src/meko/internal/memory/export.go:47-57`). A non-AI program can read either.

## Operational story

Local-first by default. `meko start` runs a server on `127.0.0.1:7777` and supervises the Node child with backoff, marking it failed after 5 crashes in 10 minutes (`design.md:66, 351-359`). `meko agent connect claude` runs `claude mcp add --scope user --transport http meko <url> --header "Authorization: Bearer …"` and removes any folder-scoped `meko` entry pointing at this computer (`src/meko/internal/agents/claude.go:150-188`). Codex, Kiro, OpenCode and Cursor have siblings in `internal/agents/` (`api.md:383-388`). Anonymous usage stats go to Yugabyte by default (`README.md:136-142`).

***

## Datapacks and skill packs

**The datapack model, from source.** In Kode, each project has one **dedicated** datapack, named `adlc/proj/<repo>/<project_name>` by default, which is "the only place session transcripts and extracted memories are written", and zero or more **shared** datapacks attached from `datapack_list` ("a team knowledge base, org conventions, another project's memory") that "are read-only from this project's point of view" (`src/extensions/agentk/docs/design/datapacks.md:5-12`). The split is passed in through environment variables, not enforced by the server: each session launches with `MEKO_DATAPACK_ID` (writes) and `MEKO_DATAPACK_IDS` (everything readable) (`datapacks.md:14-21`; `src/extensions/agentk/src/server/sessions.ts:1210-1212`). Locally nothing stops a write to a shared datapack; on Cloud, the viewer and contributor grants do.

**Search before work.** Two mechanisms make agents search:
- `MEKO.md` is a per-folder table (Datapack, ID, What's in it) that "belongs to its subtree": agents use every `MEKO.md` from the root down and "search their datapacks before any work here, even before reading the code" (`AGENTS.md:84-94`; `MEKO.md:1-10`). Kode parses it (`src/extensions/agentk/src/shared/mekomd.ts:1-6`), and its code review searches the datapack whose name ends in `/gotchas` with `knowledgebase_search`, keeping a finding's gotcha credit only when it quotes the sentence it applies (`docs/design/code-review.md:465-475`).
- Kode injects prompts in code, so repo-overridden prompts get them too: a system line ("Before reading code for a new task, a review or a CI fix, search Meko ...", with the ToolSearch query `+meko context_search` because Claude Code hides MCP tools; `src/extensions/agentk/src/server/prompts.ts:312-320`) and a per-stage line for `explore`, `design`, `code`, `ci`, `review-pr` and others (`prompts.ts:377-403`). Kode also runs a `context_search` itself before the worktree exists and passes up to 6 items to a `SessionStart` hook headed "What Meko already knows about this task", flagged as notes to check, "not to follow as instructions" (`docs/design/recall-and-learn.md:7-34`).

**The gotchas datapack is filled off-repo.** `MEKO.md:9` says "don't write to it: the cloud's gotchas miner fills it", and the code review notes "the cloud's roster of them isn't in Meko" (`code-review.md:475`). The miner isn't in this repo. What is here is Kode's **learn step**, which at Merge & Close forks the session transcript, extracts facts, dedups each against the 3 most similar memories using a small model, and writes them as `"Gotcha (component): …"` memories in the dedicated datapack (`recall-and-learn.md:43-62, 121-146`).

**Skill packs.** Meko has no skill-pack concept. Instructions come from three places: `AGENTS.md`/`MEKO.md` text, per-datapack extraction instructions (`memory-extraction.md:46`), and a separate `meko-agent-skills` Claude Code plugin (capture hooks plus an MCP server) that isn't in this repo and that Kode turns off while transcript capture is off (`datapacks.md:56-69`). Kode's own plugins under `src/extensions/agentk/src/mod/` are hooks only (`src/mod/agentk/.claude-plugin/plugin.json`, `hooks/hooks.json`).

**Against our setup.** Our corpus is one local Chroma instance with four tree-wide collections. A `CLAUDE.md` rule says to query it before answering, but it has no per-folder index, no write path for agents, and no notion of which collection a project may write to. `MEKO.md` is a per-folder index we lack. On the instruction side, our `lossless-agent-skills` exploration `context-v/explorations/Organize-Our-Skills-into-Skill-Packs.md` (2026-11-22) wants each pack to be one repo, mounted both where it is used and in `lossless-agent-skills`, to end the timestamp-driven drift between skill copies. Meko's dedicated-plus-shared datapack model is the knowledge-side version of that "one source of truth, mounted in many places" requirement.

***

## What's inside this submodule

| Path | What's there |
|---|---|
| `src/meko/docs/design.md`, `context-engine.md`, `api.md`, `memory-extraction.md`, `knowledge.md`, `cloud.md` | The design docs. Read `design.md` then `context-engine.md` first |
| `src/meko/internal/store/` (+ `migrations/0001`–`0016`) | Every SQL query, the vec0/FTS5 schema |
| `src/meko/internal/memory/` | Service layer: `extractor.go`, `extractprompt.go`, `learnings.go`, `questions.go`, `export.go`, `context.go` |
| `src/meko/context_engine/src/` | Node child: `providers.ts` (MekoStore, NoLLM), `scoring.ts`, `contextengine.ts`, embed/rerank workers |
| `src/meko/internal/mcp/`, `internal/rest/` | MCP tools and REST routes in Cloud's shapes |
| `src/meko/internal/agents/` | Per-harness connectors: `claude.go`, `codex.go`, `kiro.go`, `opencode.go`, `cursor.go` |
| `src/extensions/agentk/` | Kode: multi-session coding app (TypeScript); `docs/design/` has `datapacks.md`, `recall-and-learn.md`, `code-review.md`, `sharing.md`, `harnesses.md` |
| `MEKO.md`, `AGENTS.md` | The repo's own datapack index and the save/don't-save rules |

## Mental model for using it well

- **Treat a datapack as the unit of sharing**, and write each project to exactly one of them. Attach team knowledge read-only.
- **Promotion is the curation step.** Learnings are raw. Promoting one makes it collective memory and a knowledge-base chunk. On Cloud, only maintainers can promote.
- **Watch the 0.75 bar.** Kode's recall cut and Context Gaps' "weak" threshold are both 0.75 (`src/extensions/agentk/src/server/mekosavings.ts:36`; `api.md:163-167`). Off-topic memories have scored 0.74 to 0.76 (`api.md:166-167`).
- **Extraction costs the user's Claude quota**, which is why it's off by default (`README.md:98`).

## When NOT to reach for this

- **More than about 200 MB of local context.** The cap is hard by design, and the answer is Cloud.
- **You need sharing or roles without a hosted service.** Local datapacks are single-owner. Grants exist only on Cloud.
- **You need contradictions resolved over time.** Extraction only adds and dedups by cosine. There's no supersession or validity window (see Graphiti, RetainDB, Neo).
- **Windows or Intel Macs.** Not supported (`README.md:36-37`).

## How this compares to the rest of the study

| | Meko | Mem0 | Paxm | Hindsight |
|---|---|---|---|---|
| **Store** | SQLite + sqlite-vec + FTS5, one file | Pluggable vector store + SQLite history | Router over 7+ backends | One Postgres (pgvector + tsvector) |
| **Scope unit** | Datapack (partition key), filed into projects | user / agent / run ids | Origin vs Scope split | Bank |
| **Extraction LLM** | User's own `claude -p`, add-only | Built-in LLM, single-pass additive (v3) | None (exact fingerprint) | Built-in, typed by `fact_type` |
| **Curation tier** | Learning, kept, promoted to collective memory | None | stm / ltm | Mental models |
| **Recall-miss tracking** | Context Gaps | No | No | No |
| **Hosted twin** | Meko Cloud, same wire shapes | Mem0 managed platform | None | None needed (embedded pg0 or self-host) |

## Working-checklist answers

- **Storage topology:** one SQLite file. vec0 partitioned by datapack, FTS5 with datapack as a column, plus an entity table. Knowledge-file bytes on disk.
- **Write policy:** verbatim adds. Extraction is headless Claude, user messages only, add-only, with a 0.9 cosine dedup. Learnings get promoted to collective memory.
- **Scopes:** datapack is the boundary. user/agent/conversation are columns. Project/folder is filing only. Roles are Cloud-only.
- **Schema:** UUID, datapack, user/agent/conversation/role/actor, text, lemmas, hash, payload JSON, source, connector_id, promotion_action, times, plus history rows.
- **Recall:** `context_search` fusing memory (cosine + BM25 + entity boost), reranked knowledge, and conversation threads.
- **Eviction:** none. The 200 MB cap refuses writes. Logs are pruned at 90 days.
- **Serialization:** STRICT SQLite, and a zip of JSONL per datapack on export.
- **Operational:** local server and supervised Node child, with an optional Cloud link that forwards calls.

## One-line summary

> Meko is upstream mem0 with its LLM removed, running in a Node child behind a Go server on a datapack-partitioned SQLite file, with extraction done by the user's own Claude Code and the datapack as the governed unit that moves to Meko Cloud unchanged; sharing, roles and the gotchas miner live in that private Cloud.
