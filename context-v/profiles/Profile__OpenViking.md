---
name: OpenViking Profile
slug: openviking
upstream: https://github.com/volcengine/OpenViking
package: openviking (PyPI), @openviking/cli (npm), ov_cli (Cargo), ghcr.io/volcengine/openviking (Docker)
license: AGPL-3.0 (main); Apache-2.0 (crates/ov_cli, examples); MIT (npm CLI shim)
maintainer: Volcengine / ByteDance (Beijing Volcano Engine Technology Co., Ltd.)
study: studies/memory-layers-for-agents
profile_path: studies/memory-layers-for-agents/openviking
profile_kind: Rust core + Python service + multi-language CLI + HTTP server + WebDAV + Helm chart
date_created: 2026-05-26
site_uuid: ba2be26e-a6d5-46c7-83fc-7fbf80ce117a
hex_code: xf3nv5
date_authored_initial_draft: 2026-05-26
date_authored_current_draft: 2026-11-23
lede: >-
  Memories, resources, and skills collapse into one `viking://` tree. Retrieval
  used to walk that tree recursively; since v0.4.23 it is one global vector
  search scoped by it.
summary: >-
  Profile of OpenViking (Volcengine / ByteDance) as pinned in the
  memory-layers-for-agents study. It is the study's strongest context-as-
  filesystem bet and its most production-shaped deployment story. Covers the
  Rust RAGFS core with the Python service layer above it, the L0/L1/L2 contract
  and the vector index as a derived secondary index, context_type derived from
  the URI, the IntentAnalyzer plus a global scoped vector search (the
  priority-queue tree descent was removed in September 2026), the async
  bottom-up semantic queue, session commit with typed memory extraction and its
  memory_diff.json audit log, calendar path variables, the account/user/peer
  identity tree, the Rust path-lock transaction model, and the newer Gateway
  and `ov compile` surfaces. Note the license split: AGPL-3.0 on the server.
publish: true
title: OpenViking — Profile
date_modified: 2026-11-23
authors:
  - Michael Staton
status: Reference
type: Profiles
---

# OpenViking — Profile

A profile of OpenViking as it lives in this study (`studies/memory-layers-for-agents/openviking/`). Cites pinned paths so you can jump to source rather than trust paraphrase. The closest comparators in the study are [[Profile__Letta]] (git-backed memory mode renders memory blocks as files — same filesystem instinct, much flatter layout) and [[Profile__Graphify]] (also unifies resource/memory into one substrate, but as a knowledge graph instead of a tree). Read alongside [[Profile__Mem0]] for contrast on retrieval shape, and [[Profile__StateBench]] for what we'd want to measure.

## TL;DR

OpenViking is Volcengine/ByteDance's open-source **Context Database for AI Agents** — a Rust-cored, Python-orchestrated, HTTP-fronted system that bets on **"context-as-filesystem"** instead of flat vector storage. Everything an agent might know (`resources/`, `user/{user_id}/memories/`, `user/{user_id}/skills/`, `user/{user_id}/peers/`, `user/{user_id}/sessions/`, account-shared `agent/skills/`) is unified into one virtual filesystem addressed by `viking://` URIs (`docs/en/concepts/04-viking-uri.md`), and every directory carries three abstraction tiers: **L0** (abstract, 256-character default budget), **L1** (overview, 4000-character default budget), **L2** (the actual files), stored as `.abstract.md`, `.overview.md`, and the raw files respectively (`docs/en/concepts/03-context-layers.md:5-13`).

The engine is split: a Rust crate `ragfs` (`crates/ragfs/`) provides the hierarchical filesystem abstraction (a Rust port of the Go AGFS originally by c4pt0r; see `crates/ragfs/ORIGIN.md`), the Python `openviking/` package layers parsing/extraction/retrieval/session over it (`docs/en/concepts/01-architecture.md`), and a separate Rust `ov_cli` (`crates/ov_cli/`) talks to the server over HTTP. A FastAPI HTTP service on port 1933 exposes the whole thing (`docs/en/api/01-overview.md`); Docker, Helm, and Caddy ship with the repo.

Retrieval **used to be** directory-recursive: a priority-queue walk descended from scope roots, scoring each subdirectory's L0/L1 and recursing into the winners until convergence. That was the bet the earlier version of this profile led with. In September 2026 (PR #5450, first released in v0.4.23 on 2026-10-02) the walk was removed. `HierarchicalRetriever` now runs one global vector search per query, constrained by target directory, context type, ACL, metadata filters, and `level`, then reranks once (`docs/en/concepts/07-retrieval.md:80-106`, `openviking/retrieve/hierarchical_retriever.py:66`). The tree still matters, as the scope filter and as the agent's deterministic `ls`/`tree`/`read` surface, but it no longer drives ranking.

If you want one sentence: **OpenViking is a Rust-fast virtual filesystem with L0/L1/L2 tiers per directory, unified URIs for memories/resources/skills, tree-scoped vector retrieval, and a production-grade Volcengine deployment story; context engineering treated as filesystem engineering.**

Citations in this profile are to the pin `b48e89da5928` (2026-11-23). OpenViking rewrote its README and most concept docs between pins, so most doc citations are now file-level rather than line-level.

## Why this exists — the design bet

The README's "Why OpenViking" section (`README.md:57-62`) states the bet in four bullets: one filesystem for knowledge, memory, and skills; search a directory, not the whole index; read the summary before the source; sessions become files you can read.

1. **Reject flat vector storage as the substrate of record.** The README frames ordinary agent memory as "a black box: text goes in, embeddings come out" (`README.md:46`). The vector index still exists, but only as a *secondary index* over the filesystem of record: RAGFS holds the bytes, the vector DB holds references (`docs/en/concepts/05-storage.md`, `docs/en/concepts/09-transaction.md:7`).
2. **Unify memory, resource, and skill into one substrate.** Three concepts other systems treat as separate stores ([[Profile__Mem0]]'s entity store, [[Profile__Volt]]'s bindles, [[Profile__Letta]]'s archival memory) collapse here into one tree: `viking://~/memories/`, `viking://resources/{project}/`, `viking://~/skills/` and `viking://agent/skills/`, `viking://~/memories/cases/` (`docs/en/concepts/04-viking-uri.md`, `docs/en/concepts/02-context-types.md`). All are just nodes; the same `ls`, `tree`, `find`, `grep` operations work on all of them.
3. **Layered loading, not pre-summarization.** L0/L1 are directory-level sidecars co-located with the leaf files, and the agent loads the cheapest tier first (`docs/en/concepts/03-context-layers.md:13`). Since v0.5 the contract is looser: L0 and L1 may exist independently, and `mkdir()` creates only L0 at first (`docs/en/concepts/03-context-layers.md:15`).
4. **Filesystem ops as the agent's verb surface.** `ls`, `tree`, `read`, `write`, `grep`, `glob`, `find` (`docs/en/api/01-overview.md`). An agent navigates the corpus the way a developer navigates a repo: deterministic paths first, semantic search as a fallback. This is the same instinct as [[Profile__Letta]]'s git-backed memory mode, but generalized from "memory blocks" to "everything."
5. **Retrieval is scoped by the tree.** The earlier claim here was that retrieval trajectories through the tree were observable artifacts. With the recursive walk removed, there is no trajectory to observe; what remains is that a search can be bounded to one subtree instead of the whole index (`README.md:60`).
6. **Rust where it has to be fast, Python where it has to be flexible.** The filesystem and path locks are Rust (`crates/ragfs/`), the agent-facing CLI is Rust (`crates/ov_cli/`), but parsing, semantic generation, session compression, memory extraction (anything that calls an LLM) is Python (`openviking/parse/`, `openviking/session/`, `openviking/retrieve/`).

## Storage topology

| Layer | Backend | Purpose | Code |
|---|---|---|---|
| **Filesystem (source of truth)** | RAGFS (Rust port of AGFS) with pluggable backends (`localfs`, `s3fs`, `memfs`, `kvfs`, `queuefs`, `sqlfs`, `serverinfofs`); optional multi-write mode with async backup backends | L0/L1/L2 bytes, `.meta.json`, multimedia | `crates/ragfs/src/plugins/`, `docs/en/concepts/05-storage.md:49-70`, `docs/en/concepts/14-multi-write-storage.md` |
| **VikingFS (URI abstraction)** | Python layer over RAGFS, now a package | Translates `viking://...` URIs to per-account filesystem paths; expands the `~` home alias; enforces scope and ACL rules | `openviking/storage/viking_fs/` (`_base.py`, `_ops.py`, `_access.py`, `_vector.py`, ...), `docs/en/concepts/05-storage.md:24-47` |
| **Vector index (secondary index)** | Pluggable: `local`, `http`, `volcengine` VikingDB, VikingDB private, `pgvector`, openGauss | URIs + dense vector + sparse vector + abstract text + scope and ACL metadata | `openviking/storage/vectordb_adapters/`, `docs/en/concepts/05-storage.md:83-` |
| **Embedding queue** | `queuefs` (Python + Rust plugin) | Bottom-up async L0/L1 generation after writes; also carries durable `session_commit` jobs | `openviking/storage/queuefs/semantic_processor.py`, `crates/ragfs/src/plugins/queuefs/`, `docs/en/concepts/16-queue-lifecycle.md` |
| **Path locks** | Rust `PathLockManager` in RAGFS, reached from Python through the binding | Write-exclusive operations across FS + VectorDB + queue | `crates/ragfs-python/src/binding/client/coordination.rs`, `openviking/storage/transaction/__init__.py`, `docs/en/concepts/09-transaction.md:27-29` |
| **Operational stores** | API-key store, upload-token store, OAuth cache; optional SQLite/MySQL/PostgreSQL task and watch stores | Multi-tenant identity, OAuth tokens, task state | `openviking/server/api_keys/`, `openviking/server/oauth/`, `openviking/storage/sql_runtime.py` |

The dual-store discipline, "FS is the source of truth and VectorDB is a derived index" (`docs/en/concepts/09-transaction.md:7`), is load-bearing. The transaction model is explicitly biased toward *consistency over recall* ("Better to miss a search result than to return a bad one", `docs/en/concepts/09-transaction.md:9`).

## Schema of a node — the L0/L1/L2 contract

A semantically processed directory looks like this (`docs/en/concepts/03-context-layers.md`, `README.md:98-107`):

```
viking://resources/docs/auth/
├── .abstract.md          # L0: abstract, 256-character default budget
├── .overview.md          # L1: overview, 4000-character default budget
├── .meta.json            # Node metadata
├── oauth.md              # L2: full content
├── jwt.md                # L2
└── api-keys.md           # L2
```

L0 and L1 are directory-level sidecars; ordinary files do not get their own (`docs/en/concepts/03-context-layers.md:13`). The `.relations.json` sidecar and the Relations API that earlier versions of this profile described were removed in August 2026 (PR #3956, "remove resource relation edges"). The vector-index payload for *each* record (`openviking/storage/collection_schemas.py:88-146`):

```python
{
    "id": "string (primary key)",
    "uri": "viking://...",
    "type": "string (reserved for file/dir/image/repo)",
    "context_type": "resource|memory|skill",   # derived from URI
    "level": 0 | 1 | 2,                         # L0/L1/L2 tier
    "vector": <dense embedding>,
    "sparse_vector": <BM25-style sparse>,
    "abstract": "<L0 text inlined for rerank>",
    "content": "<text>", "md5": "<stored bytes hash>",
    "name": "...", "description": "...", "tags": "...", "search_tags": [...],
    "account_id": "...", "owner_user_id": "...", <ACL mode + grant lists>,
    "created_at": "...", "updated_at": "...",
    "active_count": int64,                      # usage counter
}
```

`owner_agent_id` is gone from the schema; ACL fields are new (`openviking/storage/collection_schemas.py:132-145`). The `context_type` is **derived from the URI** (`openviking/storage/collection_schemas.py:94-100`):

- URI under a user skills directory → `"skill"`
- URI contains `"memories"` → `"memory"`
- everything else → `"resource"`

This is the same-tree-different-prefix bet: the storage doesn't care, but retrieval can scope cleanly. Compare to [[Profile__Mem0]]'s entity-store-as-separate-collection or [[Profile__Graphify]]'s graph-edge typing; OpenViking puts the type *in the path*.

## Recall API — tree-scoped global search

There are two retrieval entry points (`docs/en/concepts/07-retrieval.md:11-21`):

| Verb | LLM intent analysis | Session context | Use case |
|---|---|---|---|
| `find(query, target_uri=...)` | No | Not used | Simple lookup |
| `search(query, session_id=...)` | Yes, when enabled and the session has content (zero or more typed queries) | Optional | Complex agent task |

`search()` can run an `IntentAnalyzer` LLM call that turns one user query into zero or more `TypedQuery` records (`docs/en/concepts/07-retrieval.md:46-79`). `skill` queries get verb-first phrasing ("Create RFC document"), `resource` queries get noun phrases ("RFC document template"), `memory` queries get "User's XX" phrasing. The planner model is separately configurable (`query_planner`, falling back to `vlm`).

Then `HierarchicalRetriever` (`openviking/retrieve/hierarchical_retriever.py:66`) runs **one global vector search per query**. Target directories, context type, permissions, metadata filters, and `level` constrain the scope; without an explicit `level`, a text query can match L0, L1, or L2 records directly. In the doc's own words, "retrieval does not navigate directories recursively" (`docs/en/concepts/07-retrieval.md:80-82`).

| Mode | Vector candidates | Rerank |
|---|---|---|
| QUICK (`find()`) | `limit` | Off |
| THINKING with a reranker (`search()`) | `2 × limit` | One pass, returns at most `limit` |
| THINKING without a reranker | `limit` | Off |

Rerank runs once over the completed candidate set and never starts another search (`docs/en/concepts/07-retrieval.md:96-106`). Scores use rerank scores when available, otherwise vector scores; access frequency, update time, and parent-directory scores do not affect them. The doc states plainly: "Directory priority queues, parent-to-child score propagation, and convergence rounds are removed" (`docs/en/concepts/07-retrieval.md:104`). Newer additions on this path: image and multimodal hybrid queries (PRs #3093, #6128), a `keywords` search type (#5456), and event time-decay ranking (#5214).

`MatchedContext` (`docs/en/concepts/07-retrieval.md:112-122`) now carries `uri`, `context_type`, `level`, `abstract`, and `score`; the `is_leaf` and `relations` fields are gone.

The filesystem verbs (`ls`, `tree`, `stat`, `read`, `write`, `grep`, `glob`; `docs/en/api/01-overview.md`) are the **other half** of retrieval, and with the recursive walk gone they are now the only way to navigate the tree deliberately. An agent that knows the answer is "somewhere in `viking://resources/my-project/docs/`" doesn't need a vector search at all; it `tree`s the directory and `read`s the leaf.

## Write policy & extraction loop

Two write paths:

**Resource ingest** (`docs/en/concepts/06-extraction.md`): `Input File → Parser → TreeBuilder → SemanticQueue → Vector Index`. Parsers are format-specific (`openviking/parse/parsers/`). The formal content tree is updated before semantic or embedding work runs (`docs/en/concepts/09-transaction.md:117-120`); the SemanticQueue then does **bottom-up async L0/L1 generation**, leaf directories first, parent L1 aggregated from children. Vector indexing happens after that, off the write path.

**Session commit** (`docs/en/concepts/08-session.md:61-118`): two-phase. Phase 1 holds an EXACT lock on the session root, archives messages to `viking://user/{user_id}/sessions/{session_id}/history/archive_NNN/`, and returns a `task_id` (no task for a no-op commit). Phase 2 runs in the background as a durable `session_commit` queue job: archive summaries, memory extraction, and a `memory_diff.json` audit log. At the new pin (listed as unreleased in the changelog), Working Memory summaries are **off by default**; commits still archive raw messages and extract long-term memories (`docs/en/about/02-changelog.md`, "Working Memory defaults off").

Built-in memory types are now `profile`, `preferences`, `entities`, `events`, `cases`, `trajectories`, and `experiences` (`docs/en/concepts/02-context-types.md`, `docs/en/concepts/08-session.md:145`). The schema-defined `memories/tools/` and `memories/skills/` types are disabled; standalone Skills live at `viking://user/{user_id}/skills/{name}/SKILL.md`. `experiences` turns on the "Agent Evolution" pipeline, which organizes a task as a case, records its trajectory, and distills reusable experiences (`docs/en/concepts/08-session.md:147-149`). A static `memories/soul.md` holds agent instructions and is never extracted or merged.

Memory extraction itself is still a `ReAct` loop (`openviking/session/memory/extract_loop.py:135`, `max_iterations: int = 3`): VLM with tools, able to read existing memories before deciding to create, merge/edit, or delete files. Memory schemas, write permissions, and output validation constrain the operations (`docs/en/concepts/08-session.md:151-155`).

The `memory_diff.json` written to each archive (`docs/en/concepts/08-session.md:157-`) is the key operational artifact. It records additions, updates, and deletions as actual file changes, plus `skipped_operations` for proposals rejected by validation or policy. Compare to [[Profile__Mem0]]'s OSS path, which records history but does not write a per-commit diff file.

The **merge operations** (`openviking/session/memory/merge_op/`: `immutable.py`, `patch.py`, `replace.py`, `sum.py`, `link_merge.py`, plus `patch_handler.py`) implement the per-type update strategy.

## Scopes & namespacing — multi-tenant by construction

URI scopes (`docs/en/concepts/04-viking-uri.md:15-30`):

| Scope | Visibility | Lifecycle |
|---|---|---|
| `resources` | Account-global, with optional per-directory/per-file ACLs | Long-term |
| `user` | Current user (memories, private resources, skills, peers, sessions) | Long-term / session lifetime |
| `agent` | Account-global skills and configuration (endpoints, tools, payments planned) | Long-term |
| `temp` / `queue` / `upload` | Internal (never API-addressable) | Transient |

The separate `session` scope is gone; sessions live at `viking://user/{user_id}/sessions/` and the old `viking://session/{id}` URI is rejected. The `~` home alias expands to the caller's own `viking://user/{user_id}` (`docs/en/concepts/04-viking-uri.md:32-53`).

Identity is now **account and user**, plus **peers** inside a user (`docs/en/concepts/11-multi-tenant.md`). The earlier `agent_id` dimension and the `isolate_agent_scope_by_user` switch no longer appear anywhere in `openviking/` or `docs/en/`. The User/Peer memory isolation model landed in June 2026 (PR #2236). A peer is a stable interaction counterpart, for example a web visitor or a code repository; the coding-agent plugins now derive a workspace's peer from its git `origin` URL (`docs/en/about/02-changelog.md`, "Workspace peer derivation"). Roles are still `ROOT` / `ADMIN` / `USER`, and a `trusted` auth mode lets an upstream gateway inject account and user headers.

Compare [[Profile__Mem0]]'s required `user/agent/run` scope triple: OpenViking treats scope as an explicit identity tree addressable up-front via the URI.

Calendar **path variables** (`docs/en/concepts/04-viking-uri.md:180-209`) are still unique among the systems in this study: `{calendar:today}`, `{calendar:ym}`, `{calendar:yq}`, `{calendar:yw}`, etc., resolved server-side at API execution time. `viking://resources/emails/{calendar:today}/inbox` lands at `viking://resources/emails/2026/05/26/inbox`. Time becomes part of the path, not just a metadata field.

## Eviction, compaction, transactional safety

**Compaction-as-archival**: `session.commit()` is the dominant compaction event (`docs/en/concepts/08-session.md:101-118`). Messages are not deleted; they roll off into `history/archive_NNN/`, so the long tail of session state stays readable without polluting the live context.

**Hotness**: `active_count` is still a per-record usage counter, and `openviking/retrieve/memory_lifecycle.py` still computes a hotness score from it. But hotness weighting was removed from retrieval ranking in September 2026 (PR #5454); the score now only feeds the memory health statistics API that classifies cold, warm, and hot memories. Compare [[Profile__Neo]]'s `confidence × success` weighting, which still drives recall.

**Memory lifecycle**: `memory_lifecycle.py` handles hotness classification; the old `openviking/session/memory_archiver.py` is gone. Retention and archive handling now sit in `openviking/session/retention.py` and `openviking/session/archive_store.py`, with a TTL subsystem in `openviking/ttl/`.

**Transactional model**: path locks moved from Python into Rust (PRs #3557, #3602). The Rust RAGFS `PathLockManager` owns runtime locks, and Python services acquire, hand off, and release leases through the binding (`docs/en/concepts/09-transaction.md:27`). EXACT locks protect one path and TREE locks a subtree (`docs/en/concepts/09-transaction.md:29`). Writes still follow reverse-order safety: for `rm`, delete the index first, then the file, so an index-deletion failure leaves the source in place (`docs/en/concepts/09-transaction.md:46-54`). The standalone **redo log is gone**: Phase 1 of a session commit persists archive metadata and enqueues a durable `SessionCommitMsg`, and on restart the QueueManager resumes leftover `session_commit` jobs (`docs/en/concepts/09-transaction.md:38-42`). Summary sidecar writes now take brief ExactPathLocks and drop stale results by `coalesce_version` instead of holding a long TreeLock (`docs/en/concepts/09-transaction.md:138-149`).

This is still more careful than anything else in the study. Compare [[Profile__Mem0]]'s lack of locking in OSS or [[Profile__Letta]]'s git-backed mode (which relies on git itself for atomicity).

## Operational story

The deployment story narrowed to **server-only** (`docs/en/concepts/01-architecture.md:109-133`):

1. **HTTP server**: `openviking-server` boots FastAPI on `:1933`. Clients are the Python SDK (`SyncHTTPClient`/`AsyncHTTPClient`, `sdk/python/`), Go and TypeScript SDKs (`sdk/go/`, `sdk/typescript/`), the Rust `ov` CLI (`crates/ov_cli/src/main.rs`), curl, or MCP at `/mcp`. A WebDAV surface sits at `/webdav/resources/{path}` (`docs/en/api/01-overview.md:610-616`). The Python **embedded mode** that earlier versions of this profile described (`ov.OpenViking(path="./data")`) was removed in August 2026 (PR #3712); `examples/quick_start.py` now starts a server and talks to it over HTTP.
2. **Docker / Helm / Cloud**: `docker-compose.yml` ships `ghcr.io/volcengine/openviking:latest` (`docker-compose.yml:22`), with the Caddy proxy on `:1934` kept as a legacy entry point (`docker-compose.yml:46-57`). Helm charts at `deploy/helm/openviking/` and, new, `deploy/helm/openviking-gateway/`. The official image bundles VikingBot and starts it by default (`README.md:325-335`).
3. **Commercial editions** (new in the README, `README.md:343-364`): a managed SaaS on Volcano Engine (BytePlus outside China is planned) and a license-key "Self-Managed" edition that adds distributed deployment. The open-source server stays AGPLv3 and needs no activation key (`README.md:339`).

Newer surfaces worth knowing:

- **OpenViking Gateway** (`openviking_gateway/`, `docs/en/gateway/`): a proxy that sits between an agent and its model API. Per turn it recognizes the conversation, replays memory blocks it added earlier exactly as before (so the prompt cache keeps hitting), recalls only for a new user message, forwards in the same API, and saves the finished turn to OpenViking (`docs/en/gateway/01-how-it-works.md`). It was called Context Gateway until PR #5795 (2026-10-09). This moves memory injection out of per-agent plugins and into the network path.
- **`ov compile`** (`docs/en/context-compilation/`): reads sources from OpenViking and uses an installed Skill to write a wiki, knowledge graph, daily report, or memory consolidation back into the tree, run by an agent runtime (VikingBot locally). Added July 2026 (PR #3567).
- **Agent Plugins 1.0 package** (`agent-plugins/`): one portable plugin (manifest, MCP server proxy to `/mcp`, and `openviking-memory` skill) for any client that implements the Agent Plugins 1.0 spec. Per-harness plugins also remain under `examples/` (`claude-code-memory-plugin/`, `codex-memory-plugin/`, `opencode-plugin/`, `openclaw-plugin/`, `hermes-plugin/`, `pi-coding-agent-extension/`, `openwebui-plugin/`, and more).
- **Resource ACLs** (`docs/en/concepts/15-acl.md`, PR #3213): opt-in per-directory and per-file grants inside the account-shared `resources` scope, enforced as vector-index filters.
- **Desktop app (beta)** for macOS and Windows that configures local agent integrations and inspects recall and capture events (`README.md:315-323`).
- **Framework adapters** moved out of the server: LangChain/LangGraph is now the separate `langchain-openviking` package under `integrations/frameworks/langchain/`, and a Microsoft Agent Framework adapter sits beside it. `openviking.integrations.langchain` imports now fail.

Model providers are still broad (Volcengine Doubao, OpenAI, Gemini, Ollama, and others; `docs/en/guides/01-configuration.md`), and `openviking-server init` still bootstraps a config. Observability: `openviking/observability/`, `/api/v1/observer/` endpoints (`docs/en/api/01-overview.md:517-`), Prometheus metrics (`docs/en/concepts/12-metrics.md`), and Grafana dashboards in `examples/grafana/`.

## Benchmarks

The README's benchmark section was rewritten (`README.md:109-123`). It now reports results for OpenViking 0.3.22, measured with Doubao 2.0 Pro as the VLM, with full setup in an external blog post. Vendor-reported numbers:

| Benchmark | Agent | Native memory | With OpenViking |
|---|---|---|---|
| LoCoMo accuracy | OpenClaw | 24.20% | 82.08% |
| LoCoMo accuracy | Hermes | 33.38% | 82.86% |
| LoCoMo accuracy | Claude Code | 57.21% | 80.32% |
| tau2-bench task success | Retail | 70.94% | 77.81% |
| tau2-bench task success | Airline | 54.38% | 66.25% |

The README also reports input tokens down 34.3–91.0% and query latency down 58.45–66.10% on LoCoMo (`README.md:122`). The earlier OpenClaw + LanceDB task-completion table that this profile quoted is no longer in the README. Note that the published numbers predate the retrieval rewrite (0.3.22 was 2026-05-29; recursive retrieval was removed in v0.4.23), so they measure the old retriever.

The harness at `benchmark/locomo/` has per-system directories for OpenViking, OpenClaw, mem0, supermemory, Claude Code, Hermes, and VikingBot; `benchmark/` also has `RAG/`, `longmemeval/`, `skillsbench/`, `tau2/`, `retrieval/`, `memory_organization/`, `vectordb_perf/`, and `custom/`. These are the system's own authors running the comparison; worth independent verification.

## What's inside this submodule

| Path | What's there |
|---|---|
| `crates/ragfs/` | **Rust filesystem core**: FileSystem trait, plugin registry, backend plugins (memfs, localfs, kvfs, queuefs, sqlfs, s3fs, serverinfofs), and now the `PathLockManager`. Rust port of Go AGFS by c4pt0r (`crates/ragfs/ORIGIN.md`) |
| `crates/ov_cli/` | **Rust CLI** (`ov` binary). `src/commands/` covers acl, admin, chat, compile, content, crypto, filesystem, mcps, observer, pack, privacy, resources, search, session, skills, snapshot, system, task, watch. The `relations` command is gone |
| `crates/ragfs-python/` | **PyO3 bindings**; the Python server reaches RAGFS and its path locks through them |
| `openviking/` | **Python service layer**: `service/`, `retrieve/` (intent + global retriever + memory lifecycle), `session/` (memory extractor, merge ops, working memory, retention, training), `parse/`, `storage/` (`viking_fs/` package, vectordb adapters, ovpack, queuefs, ACL, SQL runtimes), `server/`, `crypto/`, `privacy/`, `observability/`, `ttl/`, `connector/`, `ingest/` |
| `openviking_gateway/` | **OpenViking Gateway**: model-API proxy with recall, replay, capture, compaction, Redis or SQLite state |
| `openviking_cli/` | Python CLI and HTTP client package |
| `sdk/` | Python, Go, and TypeScript SDKs |
| `agent-plugins/` | Agent Plugins 1.0 portable package |
| `integrations/frameworks/` | LangChain/LangGraph (AGPLv3) and Microsoft Agent Framework (MIT) adapters |
| `src/` | C++ index and store extensions (CMake-built): `index/`, `store/`, `common/` |
| `npm/cli/` | npm distribution of the Rust CLI as platform-specific optional dependencies |
| `bot/` | **VikingBot**: built-in agent framework, also the default runtime for `ov compile` |
| `docs/` | English and Chinese docs. The old `docs/design/` decision records were removed (PR #5512); concepts now run 00 through 17, including encryption, privacy, multi-write storage, ACL, and queue lifecycle |
| `examples/` | Per-harness plugins (Claude Code, Codex, OpenCode, OpenClaw, Hermes, pi, Open WebUI, DSH, Magpie, and others), `multi_tenant/`, `cloud/`, `grafana/`, `k8s-helm/`, `compile/`, and `skills/` (`openviking-memory`, `ov-resources`, `ov-skills`, `ov-experience-memory`, `ov-server-operate`, `ov_dream`, ...) |
| `benchmark/` | `locomo/`, `longmemeval/`, `RAG/`, `skillsbench/`, `tau2/`, `retrieval/`, `memory_organization/`, `vectordb_perf/`, `custom/` |
| `web-studio/`, `frontend/` | Web Studio (React + Vite) for browsing and editing the URI tree; a hosted version runs at studio.openviking.ai |
| `deploy/helm/` | Helm charts for the server and the gateway |
| `third_party/` | Vendored C++ deps: `croaring`, `krl`, `leveldb-1.23`, `rapidjson`, `spdlog-1.14.1` |

If you only read three files: `docs/en/concepts/04-viking-uri.md` (the URI scheme and identity tree), `docs/en/concepts/07-retrieval.md` (what retrieval does now, and what it stopped doing), `openviking/storage/collection_schemas.py` (the vector-index schema).

## Mental model for using it well

- **Design your URI tree first.** The system is more opinionated about *paths* than about *content*. Decide what lives under `viking://resources/`, what's per-user, and what belongs to a peer before you write integration code. The path *is* the schema, and it is now also the main lever on retrieval scope.
- **Scope every search.** With global retrieval, an unscoped `find` searches the whole account index. Pass `target_uri` (and `level` when you know which tier you want) to get the benefit the tree used to give for free.
- **Lean on `ls` and `tree` before `find`.** Deterministic navigation is faster, cheaper, and more debuggable than semantic search, and it is now the only way to walk the tree deliberately.
- **Don't fight L0/L1/L2.** If your content doesn't have natural abstractive tiers, the system synthesizes them, and its choices about what goes in `.overview.md` will leak into retrieval quality. Curate the L1 for high-stakes directories.
- **Use `{calendar:today}` for time-series data.** Emails, logs, daily reports, snapshots: path variables are built for this.
- **Audit `memory_diff.json` in production.** Every session commit writes one. Wire it into your ops dashboards before you trust auto-extraction; `skipped_operations` shows what validation refused.
- **Read the changelog before upgrading.** Between these pins the project removed embedded mode, the relations API, the redo log, the agent identity dimension, recursive retrieval, hotness ranking, and the in-tree LangChain module, and turned Working Memory off by default. Several of these are marked breaking with no compatibility period (`docs/en/about/02-changelog.md`).
- **Pin RAGFS impl explicitly.** `RAGFS_IMPL=rust` for production, `RAGFS_IMPL=go` for compatibility testing (`crates/ragfs/ORIGIN.md:13-16`). `auto` defaults to Rust with Go fallback.

## When NOT to reach for this

- **You want a flat key-value memory store.** OpenViking's overhead (tree maintenance, L0/L1 generation, per-account identity) is wasted if your access pattern is "fetch by user_id." Use [[Profile__Mem0]].
- **You want a graph-first model.** OpenViking removed its relation edges in August 2026; graph output now exists only as an `ov compile` product written into the tree. Use [[Profile__Graphify]] or [[Profile__Graphiti]].
- **You chose OpenViking for tree-walking retrieval.** That mechanism is gone. If hierarchical descent is the point, you now have to build it yourself on top of `ls`/`tree` and scoped `find`.
- **You're allergic to AGPL.** The main project is AGPL-3.0 (`LICENSE`, `README.md:415-424`). Network-service-mode deployments must release server-side modifications. The Rust CLI (`crates/LICENSE`, Apache-2.0) and examples (`examples/LICENSE`, Apache-2.0) are escape hatches if you only need the client surface, but the server you're talking to inherits AGPL obligations. The LangChain adapter is AGPLv3 too; the Microsoft Agent Framework adapter and the Hermes plugin are MIT.
- **You want an in-process library.** Embedded mode is gone; you must run a server.
- **You need offline-first single-user notes.** Our own `context-vigilance` markdown discipline is dramatically lighter: directories, frontmatter, git, wikilinks. OpenViking earns its keep once you have multiple agents, multiple users, large corpora, and recall latency budgets.
- **You can't host an embedding model.** Deploying your own server needs Python 3.10+ and a model provider with both an embedding model and a VLM (`README.md:127`). Ollama works for fully local.

## How this compares to the rest of the study

| Axis | OpenViking | [[Profile__Letta]] | [[Profile__Graphify]] | [[Profile__Mem0]] | [[Profile__Volt]] |
|---|---|---|---|---|---|
| **Substrate** | Virtual filesystem (`viking://`) over RAGFS + vector index | Postgres + git-backed memory mode renders blocks as files | Knowledge graph (entity nodes + typed edges) | Vector store + entity store + SQLite | Postgres DAG (immutable + summary) |
| **Unification claim** | Memory + resource + skill in one tree | Memory blocks + archival memory | Memory + sources in one graph | Memory only; resources external | Coding sessions only |
| **Tiering** | L0/L1/L2 per directory (256 chars / 4000 chars / full) | Block-level (small in-context, archival) | Graph node properties | None (single payload) | Bindle + summary node |
| **Retrieval** | Global vector search scoped by subtree + one rerank pass | Block recall + archival vector search | Graph traversal + cypher | Hybrid: semantic + BM25 + entity boost | Bindle expansion + `lcm_grep` |
| **Write policy** | Async L0/L1 generation; typed ReAct extract loop with merge ops | Block update via LLM tool call | Edge creation/update | Single-pass additive | Verbatim append + summary |
| **Eviction** | Archive + retention/TTL; hotness used for health stats only | Block summarization | Edge weight decay | None (OSS) | Dolt evicts bindles |
| **Supersession** | Per-type merge ops with `memory_diff.json` audit | Block-level overwrite | First-class via edges | Not modeled in OSS | Immutable log + summary |
| **Multi-tenancy** | account/user/peer, ACLs on shared resources, role-based | Org/user/agent (Letta Cloud) | Workspace | user/agent/run required | Conversation-scoped |
| **Transactional model** | Rust path locks (EXACT/TREE) + durable queue jobs | Postgres ACID | Graph DB ACID | None (OSS) | Postgres ACID |
| **CLI / verb surface** | `ov ls`, `tree`, `find`, `grep`, `read`, `compile`, `acl`, `add-resource` | API + ADE UI | API + UI | `add`/`search`/`update`/`delete` | Terminal-native (`lcm_*`) |
| **Languages** | Rust (FS+CLI+locks) + Python (service, gateway) + TS/Go (SDKs, plugins) + C++ (indices) | Python | Python | Python + TS | TS |
| **License** | AGPL-3.0 (main) + Apache-2.0 (CLI/examples) | Apache-2.0 | Apache-2.0 | Apache-2.0 | Apache-2.0 |
| **Backer / scale** | Volcengine / ByteDance (production, commercial editions) | Letta Inc. | Independent | Mem0.ai | Independent |
| **Best fit** | Multi-user, large-corpus, ops-disciplined deployments | Conversational agents with editable persona/state | Knowledge-graph-shaped domains | General-purpose memory layer | Long-horizon coding sessions |

The closest *architectural* sibling is [[Profile__Letta]] (the only other system that treats memory as files), but Letta's filesystem is a thin git layer over a Postgres-backed block store, while OpenViking's filesystem *is* the substrate. The closest *unification* sibling is [[Profile__Graphify]] (also collapses resource and memory into one store), but Graphify uses a graph and OpenViking uses a tree.

## Changes since the previous pin (2026-05-26 → 2026-11-23)

Old pin `e0ce670f5cb7` (v0.3.20 + 20 commits) → new pin `b48e89da5928` (v0.5.5 + 260 commits). 2,305 commits. Releases in range: v0.3.21 through v0.3.24, v0.4.1 through v0.4.23, v0.5.0 (2026-10-09) through v0.5.5 (2026-11-13), plus many `cli@`, `react@`, `sdk/go/`, and `openviking-superagent@` package tags. License unchanged (AGPLv3 main; `crates/LICENSE` now carries the Apache-2.0 text for the CLI).

What matters for this profile's thesis:

- **Recursive retrieval removed.** PR #5450 (2026-09-29, v0.4.23) replaced the priority-queue directory descent with one global vector search per query plus one rerank pass. `docs/en/concepts/07-retrieval.md:80-106` says directory priority queues, parent-to-child score propagation, and convergence rounds are removed. This was the profile's headline claim.
- **Hotness out of ranking.** PR #5454 removed hotness weighting from retrieval; `openviking/retrieve/memory_lifecycle.py` now only feeds memory health statistics. Event time-decay ranking was added instead (PR #5214).
- **Identity model rebuilt.** User/Peer memory isolation (PR #2236, v0.4.1) replaced the agent dimension. Memories, skills, and sessions now sit under `viking://user/{user_id}/`; `viking://agent/` is account-shared skills and configuration; `viking://session/` and uid-less `viking://user/memories/` URIs are rejected; `~` is the home alias (`docs/en/concepts/04-viking-uri.md`).
- **Memory types changed.** `patterns`, `tools`, and `skills` memory types gave way to `trajectories` and `experiences` plus an Agent Evolution pipeline (`docs/en/concepts/02-context-types.md`, `docs/en/concepts/08-session.md:145-149`).
- **Transactions moved to Rust.** Path locks now live in the RAGFS `PathLockManager` (PRs #3557, #3602); the redo log is gone, replaced by durable `session_commit` queue jobs (`docs/en/concepts/09-transaction.md:27-42`).
- **Relations removed.** `.relations.json` and the Relations API went away in PR #3956 (v0.4.16).
- **Embedded mode removed.** PR #3712 (v0.4.14); OpenViking is server-only now.
- **New surfaces.** OpenViking Gateway (`openviking_gateway/`), `ov compile` context compilation (`docs/en/context-compilation/`), resource ACLs (`docs/en/concepts/15-acl.md`), an Agent Plugins 1.0 package (`agent-plugins/`), Go and TypeScript SDKs (`sdk/`), a desktop app, and commercial SaaS and self-managed editions.
- **Working Memory off by default** and **LangChain moved out** to `langchain-openviking` (both listed under Unreleased in `docs/en/about/02-changelog.md`).
- **Benchmarks replaced.** The README now cites LoCoMo and tau2-bench numbers for v0.3.22 instead of the OpenClaw LoCoMo10 table (`README.md:109-123`).

## One-line summary

> OpenViking is a Rust-cored virtual filesystem with L0/L1/L2 tiers per directory and unified `viking://` URIs that collapse memory, resources, and skills into one per-user tree. Retrieval is now a global vector search that the tree scopes rather than drives, backed by Rust path locks and durable queue jobs and a Volcengine-grade deployment story, including a model-API gateway. It is still the strongest "context-as-filesystem" bet in the study, and a good default when path determinism, multi-tenant structure, or large-corpus token economics matter more than the simplicity of a flat key-value memory.
