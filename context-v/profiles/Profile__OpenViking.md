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
date_authored_current_draft: 2026-05-26
lede: >-
  Memories, resources, and skills collapse into one `viking://` tree, and
  retrieval is a recursive directory walk, not a vector query.
summary: >-
  Profile of OpenViking (Volcengine / ByteDance) as pinned in the
  memory-layers-for-agents study. It is the study's strongest context-as-
  filesystem bet and its most production-shaped deployment story. Covers the
  Rust RAGFS core with the Python service layer above it, the L0/L1/L2 contract
  and the vector index as a derived secondary index, context_type derived from
  the URI prefix, the IntentAnalyzer plus HierarchicalRetriever priority-queue
  descent, the async bottom-up semantic queue, session commit with 8-category
  memory extraction and its memory_diff.json audit log, calendar path variables,
  the account/user/agent identity tree, and the path-lock plus redo-log
  transaction model. Read the retrieval and transaction sections first — they
  are the two places the design is genuinely ahead of the rest of the field.
  Note the license split: AGPL-3.0 on the server.
publish: true
---

# OpenViking — Profile

A profile of OpenViking as it lives in this study (`studies/memory-layers-for-agents/openviking/`). Cites pinned paths so you can jump to source rather than trust paraphrase. The closest comparators in the study are [[Profile__Letta]] (git-backed memory mode renders memory blocks as files — same filesystem instinct, much flatter layout) and [[Profile__Graphify]] (also unifies resource/memory into one substrate, but as a knowledge graph instead of a tree). Read alongside [[Profile__Mem0]] for contrast on retrieval shape, and [[Profile__StateBench]] for what we'd want to measure.

## TL;DR

OpenViking is Volcengine/ByteDance's open-source **Context Database for AI Agents** — a Rust-cored, Python-orchestrated, HTTP-fronted system that bets on **"context-as-filesystem"** instead of flat vector storage. Everything an agent might know (`resources/`, `user/memories/`, `agent/memories/`, `agent/skills/`, per-session state) is unified into one virtual filesystem addressed by `viking://` URIs (`docs/en/concepts/04-viking-uri.md:1-30`), and every node in that tree carries three abstraction tiers — **L0** (~100-token abstract), **L1** (~1-2k-token overview), **L2** (the actual content) — stored as `.abstract.md`, `.overview.md`, and the raw files respectively (`docs/en/concepts/03-context-layers.md:5-12, 109-119`).

The engine is split: a Rust crate `ragfs` (`crates/ragfs/`) provides the hierarchical filesystem abstraction (a Rust port of the Go AGFS originally by c4pt0r — see `crates/ragfs/ORIGIN.md:1-15`), the Python `openviking/` package layers parsing/extraction/retrieval/session over it (`docs/en/concepts/01-architecture.md:7-50`), and a separate Rust `ov_cli` (`crates/ov_cli/`) talks to the server over HTTP. A FastAPI HTTP service on port 1933 exposes the whole thing (`docs/en/api/01-overview.md:144-145, 338-505`); Docker, Helm, and Caddy ship with the repo.

Retrieval is **directory-recursive**, not flat: a priority-queue walk descends from `viking://resources/`, `viking://user/memories/`, `viking://agent/skills/` etc., scoring each subdirectory's L0/L1 against the query and recursing into the winners until convergence (`docs/en/concepts/07-retrieval.md:73-130`, `openviking/retrieve/hierarchical_retriever.py:46-80`). This is the architectural bet that distinguishes OpenViking from everything else in this study.

If you want one sentence: **OpenViking is a Rust-fast virtual filesystem with three abstraction tiers per node, unified URIs for memories/resources/skills, recursive tree-walking retrieval, and a production-grade Volcengine deployment story — context engineering treated as filesystem engineering.**

## Why this exists — the design bet

The README is unusually explicit about what OpenViking refuses to do (`README.md:34-57`):

1. **Reject flat vector storage as the primary substrate.** The README calls out "fragmented vector storage" as the antipattern; the system instead "abandons the fragmented vector storage model of traditional RAG and innovatively adopts a 'file system paradigm'" (`README.md:48`). The vector index still exists, but only as a *secondary index* over the filesystem of record — the AGFS/RAGFS layer holds the bytes, the vector DB holds references (`docs/en/concepts/05-storage.md:22-35`).
2. **Unify memory, resource, and skill into one substrate.** Three concepts other systems treat as separate stores ([[Profile__Mem0]]'s entity store, [[Profile__Volt]]'s bindles, [[Profile__Letta]]'s archival memory) collapse here into one tree: `viking://user/memories/`, `viking://resources/{project}/`, `viking://agent/skills/{name}/`, `viking://agent/memories/cases/` (`docs/en/concepts/04-viking-uri.md:32-66`, `docs/en/concepts/02-context-types.md:1-16`). All four are just nodes; the same `ls`, `tree`, `find`, `grep` operations work on all of them.
3. **Layered loading, not pre-summarization.** L0/L1/L2 is not "we summarized your docs." It's a contract: every directory has `.abstract.md` and `.overview.md` co-located with the leaf files, and the agent loads the cheapest tier first (`docs/en/concepts/03-context-layers.md:107-119`). Token budget is managed by *what level the agent asks for*, not by truncating mid-retrieval.
4. **Filesystem ops as the agent's verb surface.** `ls`, `tree`, `read`, `abstract`, `overview`, `grep`, `glob`, `find` (`docs/en/concepts/05-storage.md:49-62`, `docs/en/api/01-overview.md:370-399`). An agent navigates the corpus the way a developer navigates a repo — deterministic paths first, semantic search as a fallback. This is the same instinct as [[Profile__Letta]]'s git-backed memory mode, but generalized from "memory blocks" to "everything."
5. **Retrieval trajectories are observable.** Because retrieval is a recursive descent, every step is loggable; the `Visualized Retrieval Trajectory` claim in the README (`README.md:55, 712-716`) means the path through the tree is a first-class artifact, not a vector-DB black box.
6. **Rust where it has to be fast, Python where it has to be flexible.** The filesystem is Rust (`crates/ragfs/Cargo.toml:1-30`), the agent-facing CLI is Rust (`crates/ov_cli/Cargo.toml:1-30`), but parsing, semantic generation, session compression, memory extraction — anything that calls an LLM — is Python (`openviking/parse/`, `openviking/session/`, `openviking/retrieve/`).

## Storage topology

| Layer | Backend | Purpose | Code |
|---|---|---|---|
| **Filesystem (source of truth)** | RAGFS (Rust port of AGFS) with pluggable backends — `localfs`, `s3fs`, `memfs`, `kvfs`, `queuefs`, `sqlfs`, `serverinfofs` | L0/L1/L2 bytes, `.relations.json`, `.meta.json`, multimedia | `crates/ragfs/src/plugins/`, `docs/en/concepts/05-storage.md:80-101` |
| **VikingFS (URI abstraction)** | Python layer over RAGFS | Translates `viking://...` URIs to `/local/{account_id}/...` filesystem paths; enforces scope rules | `openviking/storage/viking_fs.py`, `docs/en/concepts/05-storage.md:37-78` |
| **Vector index (secondary index)** | Pluggable: `local`, `http`, `volcengine` VikingDB | URIs + dense vector + sparse vector + abstract text + scope metadata — never file content | `openviking/storage/vectordb_adapters/{local_adapter,http_adapter,volcengine_adapter}.py`, `docs/en/concepts/05-storage.md:104-141` |
| **Embedding queue** | In-process `queuefs` (Python + Rust plugin) | Bottom-up async L0/L1 generation after writes | `openviking/storage/queuefs/semantic_processor.py`, `crates/ragfs/src/plugins/queuefs/` |
| **Path locks + redo log** | File-based fencing tokens | Write-exclusive operations across FS + VectorDB + queue; crash recovery only for session-memory extract | `openviking/storage/transaction/`, `docs/en/concepts/09-transaction.md:1-80` |
| **Operational stores** | API-key store, upload-token store, OAuth cache | Multi-tenant identity, ChatGPT/Codex OAuth tokens | `openviking/server/api_keys/`, `openviking/server/oauth/`, `openviking/server/temp_upload_store.py` |

The dual-store discipline — "FS is the source of truth, VectorDB is a derived index" (`docs/en/concepts/09-transaction.md:5-8`) — is load-bearing. The transaction model is explicitly biased toward *consistency over recall* ("Better to miss a search result than to return a bad one", `docs/en/concepts/09-transaction.md:9`).

## Schema of a node — the L0/L1/L2 contract

Every directory in the tree looks like this (`docs/en/concepts/03-context-layers.md:107-119`, `docs/en/concepts/04-viking-uri.md:301-309`):

```
viking://resources/docs/auth/
├── .abstract.md          # L0: ~100 tokens, one-sentence summary
├── .overview.md          # L1: ~1-2k tokens, structure + access guide
├── .relations.json       # Cross-node links (related URIs + reasons)
├── .meta.json            # Node metadata
├── oauth.md              # L2: full content
├── jwt.md                # L2
└── api-keys.md           # L2
```

The vector-index payload for *each* node (`openviking/storage/collection_schemas.py:72-129`):

```python
{
    "id": "string (primary key)",
    "uri": "viking://...",
    "type": "string (reserved for file/dir/image/repo)",
    "context_type": "resource|memory|skill",   # derived from URI prefix
    "level": 0 | 1 | 2,                         # L0/L1/L2 tier
    "vector": <dense embedding>,
    "sparse_vector": <BM25-style sparse>,
    "abstract": "<L0 text inlined for fast rerank>",
    "name": "...", "description": "...", "tags": "...",
    "account_id": "...", "owner_user_id": "...", "owner_agent_id": "...",
    "created_at": "...", "updated_at": "...",
    "active_count": int64,                      # usage-based hotness
}
```

The `context_type` is **derived from the URI prefix** (`openviking/storage/collection_schemas.py:77-84`):

- `viking://agent/skills/...` → `"skill"`
- URI contains `"memories"` → `"memory"`
- everything else → `"resource"`

This is the same-tree-different-prefix bet: the storage doesn't care, but retrieval can scope cleanly. Compare to [[Profile__Mem0]]'s entity-store-as-separate-collection (`Profile__Mem0.md`) or [[Profile__Graphify]]'s graph-edge typing — OpenViking puts the type *in the path*.

## Recall API — recursive directory walking

There are two retrieval entry points (`docs/en/concepts/07-retrieval.md:14-37`):

| Verb | LLM intent analysis | Session context | Use case |
|---|---|---|---|
| `find(query, target_uri=...)` | No | Not needed | Simple lookup |
| `search(query, session_info=...)` | Yes (0-5 typed queries) | Required | Complex agent task |

`search()` runs an `IntentAnalyzer` LLM call that turns one user query into 0-5 `TypedQuery` records (`docs/en/concepts/07-retrieval.md:39-72`) — `skill` queries get verb-first phrasing ("Create RFC document"), `resource` queries get noun phrases ("RFC document template"), `memory` queries get "User's XX" phrasing. Zero queries is a valid output for chitchat.

Then the `HierarchicalRetriever` walks the tree (`openviking/retrieve/hierarchical_retriever.py:46-80`, `docs/en/concepts/07-retrieval.md:73-122`):

```
Step 1: Determine root directories by context_type
        MEMORY  → viking://user/memories, viking://agent/memories
        RESOURCE → viking://resources
        SKILL   → viking://agent/skills
Step 2: Global vector search to locate starting directories (GLOBAL_SEARCH_TOPK=10)
Step 3: Push starting points into a priority queue with their rerank scores
Step 4: Pop best directory, search its children, score, push winning subdirs back
Step 5: Stop when topk unchanged for MAX_CONVERGENCE_ROUNDS=3
Step 6: Convert to MatchedContext records
```

The fan-out is bounded (`openviking/retrieve/hierarchical_retriever.py:53`): `MAX_PARALLEL_CHILD_SEARCHES = 4` per request against remote vector stores. Score propagation between parent and child is controlled by `score_propagation_alpha` (default 1.0 — child's own score only; lower values blend in the parent's, encouraging deeper drilling under high-scoring directories — `docs/en/concepts/07-retrieval.md:124-130`).

Rerank is a separate stage (`docs/en/concepts/07-retrieval.md:132-160`). If a rerank model is configured (default `doubao-seed-rerank` from Volcengine), the THINKING-mode pipeline reranks the candidate set; if rerank fails or isn't configured, fall back to raw vector scores. Rerank fires *twice*: once for starting-point evaluation, once for each level of the recursive descent.

`MatchedContext` (`openviking/retrieve/hierarchical_retriever.py` types via `openviking_cli.retrieve.types`, `docs/en/concepts/07-retrieval.md:166-175`):

```python
@dataclass
class MatchedContext:
    uri: str
    context_type: ContextType  # MEMORY / RESOURCE / SKILL
    is_leaf: bool              # file vs directory
    abstract: str              # L0 inline
    score: float
    relations: List[RelatedContext]
```

The filesystem verbs (`docs/en/api/01-overview.md:370-399`) — `ls`, `tree`, `stat`, `read`, `abstract`, `overview`, `grep`, `glob` — are the **other half** of retrieval. An agent that knows the answer is "somewhere in `viking://resources/my-project/docs/`" doesn't need a vector search at all; it `tree`s the directory and `read`s the leaf. This is the most under-rated affordance of the design: deterministic navigation is always available.

## Write policy & extraction loop

Two write paths:

**Resource ingest** (`docs/en/concepts/06-extraction.md:1-100`): `Input File → Parser → TreeBuilder → SemanticQueue → Vector Index`. Parsers are format-specific (`openviking/parse/parsers/`: markdown, pdf, html, docx, epub, code, image, video, audio, feishu, zip). Parsing produces a temp tree of files; TreeBuilder moves it under `viking://resources/{project}/`; the SemanticQueue then does **bottom-up async L0/L1 generation** — leaf nodes first, parent L1 aggregated from child L0s. Vector indexing happens after that. The parser never calls an LLM (`docs/en/concepts/06-extraction.md:15`); the semantic processor does, async, off the write path.

**Session commit** (`docs/en/concepts/08-session.md:97-135`): two-phase. Phase 1 (sync): increment compression index, write messages to `history/archive_NNN/messages.jsonl`, clear current messages, return a `task_id`. Phase 2 (async): generate `.abstract.md` and `.overview.md` for the archive, run **8-category memory extraction** (profile, preferences, entities, events, cases, patterns, tools, skills — `docs/en/concepts/08-session.md:139-148`), write a `memory_diff.json` audit log capturing every add/update/delete, mark `.done`.

Memory extraction itself is a `ReAct` loop (`openviking/session/memory/extract_loop.py:41-80`): VLM with tools, max 3 iterations, can `ls`/`read .overview.md`/`search` to pre-fetch context before deciding `create`/`merge`/`delete` per candidate (`docs/en/concepts/08-session.md:151-170`). The deduplication is *per-existing-item* — for each candidate memory, the loop returns either `skip` / `create` / `none` at the candidate level, plus `merge` / `delete` decisions per conflicting existing memory.

The `memory_diff.json` written to each archive (`docs/en/concepts/08-session.md:172-220`) is a key operational artifact — it captures `before`, `after`, and `deleted_content` for every change, enabling audit and rollback. Compare to [[Profile__Mem0]]'s OSS path which has *no* automatic supersession; OpenViking does, and writes the diff.

The five **merge operations** (`openviking/session/memory/merge_op/`: `immutable.py`, `patch.py`, `replace.py`, `sum.py`, `link_merge.py`) implement the per-category update strategy. Compare `docs/en/concepts/02-context-types.md:57-66` — `profile` merges into one file, `preferences` and `entities` are appendable, `events` and `cases` are immutable, `patterns`/`tools`/`skills` are mergeable.

## Scopes & namespacing — multi-tenant by construction

URI scopes (`docs/en/concepts/04-viking-uri.md:16-30`):

| Scope | Visibility | Lifecycle |
|---|---|---|
| `resources` | Global within account | Long-term |
| `user` | User-isolated | Long-term |
| `agent` | Agent-isolated (configurable) | Long-term |
| `session` | Current session only | Session lifetime |
| `temp` / `queue` | Internal (never API-addressable) | Transient |

Three-level identity (`docs/en/concepts/11-multi-tenant.md:21-46`): `account_id` (tenant/team/customer), `user_id` (per-account user), `agent_id` (agent within account). Roles are `ROOT` / `ADMIN` / `USER`.

The agent-URI shape is a *policy decision* per account (`docs/en/concepts/04-viking-uri.md:228-233`):

- `isolate_agent_scope_by_user = false`: `viking://agent/{agent_id}/...` (multi-user agents share the same memory)
- `isolate_agent_scope_by_user = true`: `viking://agent/{agent_id}/user/{user_id}/...` (per-user memory per agent)

This is more nuanced than [[Profile__Mem0]]'s required `user/agent/run` scope triple — OpenViking treats scope as an explicit identity tree with configurable nesting, addressable up-front via the URI.

Calendar **path variables** (`docs/en/concepts/04-viking-uri.md:117-189`) are unique among the systems in this study: `{calendar:today}`, `{calendar:ym}`, `{calendar:yq}`, etc., resolved server-side at API execution time. Lets you write `viking://resources/emails/{calendar:today}/inbox` and have it land at `viking://resources/emails/2026/05/26/inbox`. Time becomes part of the path, not just a metadata field.

## Eviction, compaction, transactional safety

**Compaction-as-archival**: `session.commit()` is the dominant compaction event (`docs/en/concepts/08-session.md:97-135`). Messages are not deleted — they roll off into `history/archive_NNN/`, with `.abstract.md` and `.overview.md` generated for the archive so the long-tail of session state remains semantically searchable but doesn't pollute the live context.

**Hotness-based ranking**: `active_count` is a per-node usage counter (`openviking/storage/collection_schemas.py:89`, `openviking/retrieve/memory_lifecycle.py:hotness_score`); the retriever blends a `hotness_alpha`-weighted hotness score into final rankings, biasing recall toward what's been used. Compare [[Profile__Neo]]'s `confidence × success` weighting.

**Memory lifecycle**: a dedicated `memory_lifecycle.py` (`openviking/retrieve/memory_lifecycle.py`) and `memory_archiver.py` (`openviking/session/memory_archiver.py`) handle aging and deduplication separately from session commit. Deeper than `Profile__Mem0`'s OSS path (none).

**Transactional model**: explicit two-primitive design — **path locks** (`openviking/storage/transaction/path_lock.py`) and **redo log** (`openviking/storage/transaction/redo_log.py`). Path locks are fencing-token-based with EXACT and TREE lock modes (`docs/en/concepts/09-transaction.md:40-72`), automatic stale-lock detection, and a background cleanup. Writes always go FS-first or VectorDB-first depending on the operation, with reverse-order safety: for `rm`, delete the index first, then the file — index deletion failure leaves both intact ("Better to miss a search result than to return a bad one"). The redo log only covers session-memory extract because that's the only non-idempotent, network-LLM-bound write.

This is significantly more careful than anything in the study. Compare [[Profile__Mem0]]'s "no eviction in OSS" or [[Profile__Letta]]'s git-backed mode (which relies on git itself for atomicity).

## Operational story

Three deployment modes, one API surface (`docs/en/concepts/01-architecture.md:123-156`):

1. **Embedded** — `client = ov.OpenViking(path="./data"); client.initialize()` (`examples/quick_start.py:3`). Auto-starts the AGFS/RAGFS subprocess in-process; uses local vector index; singleton pattern.
2. **HTTP server** — `openviking-server` boots FastAPI on `:1933` (`docs/en/api/01-overview.md:90, 144`). Python SDK (`SyncHTTPClient`, `AsyncHTTPClient`), Rust `ov` CLI (`crates/ov_cli/src/main.rs`), curl, or any HTTP client. WebDAV mount surface available at `/webdav/resources/{path}` (`docs/en/api/01-overview.md:493-503`). The `/metrics` endpoint exposes Prometheus.
3. **Docker / Helm / Cloud** — `docker-compose.yml` ships the image (`ghcr.io/volcengine/openviking:latest`) plus a Caddy reverse proxy on `:1934` (`docker-compose.yml:15-59`, `Caddyfile:1-25`). Helm chart at `deploy/helm/openviking/`. Volcengine ECS / VeLinux is the recommended cloud target (`README.md:601-606`).

Multi-language client surface:

- Python: `openviking` (`openviking/__init__.py:1-80`) — embedded `SyncOpenViking`/`AsyncOpenViking` and HTTP `SyncHTTPClient`/`AsyncHTTPClient`.
- Rust CLI: `crates/ov_cli/` ships as `ov` binary, distributed via `cargo install --git` or as platform-specific npm packages (`npm/cli/package.json:13-19`: `@openviking/cli-{darwin,linux,win32}-{arm64,x64}`).
- TypeScript editor plugins: `examples/openclaw-plugin/`, `examples/claude-code-memory-plugin/`, `examples/codex-memory-plugin/`, `examples/opencode-memory-plugin/`, `examples/opencode/plugin/`.
- LangChain/LangGraph: `openviking/integrations/langchain/`, `examples/langchain-langgraph/`.
- Web Studio: a React/Vite SPA at `web-studio/` served from `/studio` on the same port as the API.
- VikingBot: a built-in chat agent (`bot/vikingbot/`) — `openviking-server --with-bot` enables `/chat` and `/chat/stream` endpoints.

Model provider story is unusually broad (`README.md:99-269`): Volcengine (Doubao), OpenAI, OpenAI-Codex (OAuth-flow, no API key needed if logged into ChatGPT), Kimi Coding, GLM Coding Plan, Gemini, Ollama (local). The `openviking-server init` wizard auto-detects environment and pulls Ollama models for fully-local setups.

Observability is first-class: `openviking/observability/`, the `/api/v1/observer/` endpoints (queue, vikingdb, models, lock, retrieval, system — `docs/en/api/01-overview.md:443-451`), Prometheus metrics export, Grafana dashboards in `examples/grafana/`. The retrieval trajectory is observable end-to-end.

## Benchmarks — the OpenClaw integration

The headline number lives in `README.md:608-628`. On LoCoMo10 (1,540 cases) with `seed-2.0-code`:

| Configuration | Task completion | Input tokens |
|---|---|---|
| OpenClaw (built-in memory) | 35.65% | 24.6M |
| OpenClaw + LanceDB | 44.55% | 51.6M |
| **OpenClaw + OpenViking (native memory enabled)** | **51.23%** | **2.1M** |
| **OpenClaw + OpenViking (native memory disabled)** | **52.08%** | **4.3M** |

Versus the no-memory baseline: **49% improvement in task completion with 83% fewer input tokens**. Versus LanceDB: **17% improvement with 92% fewer input tokens**. The token reduction is the L0/L1/L2 design earning its keep — agents load `.abstract.md` first, only descend to `.overview.md` and L2 when needed.

The benchmark harness at `benchmark/locomo/` covers OpenViking, OpenClaw, mem0, supermemory, Claude Code, and Hermes (`benchmark/locomo/README.md:7-30`); also `benchmark/RAG/`, `benchmark/skillsbench/`, `benchmark/tau2/`, `benchmark/custom/` for additional shapes.

This is one of the most carefully presented benchmark comparisons in the study — comparable in rigor to [[Profile__StateBench]] but executed by the system's own authors. Worth independent verification.

## What's inside this submodule

| Path | What's there |
|---|---|
| `crates/ragfs/` | **Rust filesystem core** — `core/` (FileSystem trait, types, errors, plugin registry), `plugins/` (memfs, localfs, kvfs, queuefs, sqlfs, s3fs, serverinfofs), `server/` (axum HTTP), `shell/` (interactive). Rust port of Go AGFS by c4pt0r (`crates/ragfs/ORIGIN.md`) |
| `crates/ov_cli/` | **Rust CLI** (`ov` binary) — `commands/` covers resources, fs, search, session, content, observer, watch, task, admin, crypto, privacy, pack, relations |
| `crates/ragfs-python/` | **PyO3 bindings** so the Python `openviking` package can embed RAGFS in-process (no HTTP server needed for embedded mode) |
| `openviking/` | **Python service layer** — `service/` (FS/Search/Session/Resource/Relation/Pack/Debug), `retrieve/` (intent + hierarchical retriever + memory lifecycle), `session/` (compressor, memory extractor, 8-category merge ops, skill dedup), `parse/` (parsers for md/pdf/html/code/image/video/audio/epub/excel/word/feishu/zip + resource detectors), `storage/` (VikingFS, vectordb adapters, ovpack, queuefs, transaction/path-locks), `server/` (FastAPI routers + auth + OAuth), `crypto/`, `privacy/`, `observability/`, `eval/`, `integrations/langchain/`, `prompts/`, `web_studio/`, `pyagfs/` |
| `openviking_cli/` | Python CLI shim that mirrors the Rust CLI for environments without `ov` |
| `src/` | C++ embedding-engine extensions (CMake-built) — `index/`, `store/`, `common/` |
| `npm/cli/` | npm distribution of the Rust CLI as platform-specific optional dependencies |
| `bot/` | **VikingBot** — built-in chat agent (LangGraph-style) that can be started with `openviking-server --with-bot` |
| `docs/` | English (`en/`), Chinese (`zh/`), and `design/` (decision records: memory-extractor-optimization, memory-isolation, multi-tenant, parser-two-layer-refactor, openclaw-plugin, mcp-oauth2, etc.) |
| `examples/` | Concrete integrations — `claude-code-memory-plugin/`, `codex-memory-plugin/`, `opencode-memory-plugin/`, `openclaw-plugin/`, `langchain-langgraph/`, `multi_tenant/`, `cloud/`, `grafana/`, `k8s-helm/`, `skills/` (sample skill packs: `ov-add-data`, `ov-search-context`, `ov-server-operate`, `ov_dream`) |
| `benchmark/` | `locomo/`, `RAG/`, `skillsbench/`, `tau2/`, `custom/` — six-system head-to-head harness |
| `web-studio/` | React + Vite SPA — visual browser for the URI tree, served from `/studio` |
| `deploy/helm/openviking/` | Helm chart for Kubernetes deployment |
| `docker/` | Container entrypoint + pending-health server |
| `third_party/` | Vendored C++ deps: `croaring`, `krl`, `leveldb-1.23`, `rapidjson`, `spdlog-1.14.1` |

If you only read three files: `docs/en/concepts/04-viking-uri.md` (the URI scheme), `openviking/retrieve/hierarchical_retriever.py` (the recursive retrieval), `openviking/storage/collection_schemas.py` (the vector-index schema). The architectural bet is all three at once.

## Mental model for using it well

- **Design your URI tree first.** The system is more opinionated about *paths* than about *content*. Decide what lives under `viking://resources/`, what's per-user, what's per-agent, before you write a line of integration code. The path *is* the schema.
- **Lean on `ls` and `tree` before `find`.** Deterministic navigation is faster, cheaper, and more debuggable than semantic search. The agent prompts in `examples/skills/ov-search-context/` show the pattern: read `.overview.md` first, then descend.
- **Don't fight L0/L1/L2.** If your content doesn't have natural abstractive tiers, the system synthesizes them — but its choices about what's in `.overview.md` vs L2 will leak into retrieval quality. Curate the L1 for high-stakes nodes.
- **Use `{calendar:today}` for time-series data.** Emails, logs, daily reports, snapshots — path variables are designed for this and the alternative (timestamp metadata + filter) is strictly worse for the recursive-walk retriever.
- **Audit `memory_diff.json` in production.** Every session commit writes one. Wire it into your ops dashboards before you trust the auto-extraction; the 8-category extractor *does* make mistakes and the diff is the only way to see them.
- **Pin RAGFS impl explicitly.** `RAGFS_IMPL=rust` for production, `RAGFS_IMPL=go` for compatibility testing (`crates/ragfs/ORIGIN.md:13-15`). `auto` defaults to Rust with Go fallback — fine for dev, surprising in incidents.

## When NOT to reach for this

- **You want a flat key-value memory store.** OpenViking's overhead — tree maintenance, L0/L1 generation, recursive retrieval — is wasted if your access pattern is "fetch by user_id." Use [[Profile__Mem0]].
- **You want a graph-first model.** OpenViking has `.relations.json` and a Relations API, but it's a side-car on the tree, not the central data model. Use [[Profile__Graphify]] or Graphiti.
- **You're allergic to AGPL.** The main project is AGPL-3.0 (`LICENSE:1-3`, `README.md:763-770`). Network-service-mode deployments must release server-side modifications. The Rust CLI (`crates/ov_cli/LICENSE`, Apache-2.0) and examples (`examples/LICENSE`, Apache-2.0) are escape hatches if you only need the client surface — but the server you're talking to inherits AGPL obligations.
- **You don't want a Rust toolchain in your build pipeline.** Minimum Rust 1.91.1, CMake 3.12+, GCC 9+/Clang 11+ (`README.md:63-70`). Heavier than [[Profile__Mem0]]'s pip-only install.
- **You need offline-first single-user notes.** Our own `context-vigilance` markdown discipline is dramatically lighter — directories, frontmatter, git, wikilinks. OpenViking earns its keep once you have multiple agents, multiple users, large corpora, and recall latency budgets.
- **You can't host an embedding model.** OpenViking *requires* both a VLM and an embedding model to operate (`README.md:94-98`). Ollama works for fully local; nothing works fully offline.

## How this compares to the rest of the study

| Axis | OpenViking | [[Profile__Letta]] | [[Profile__Graphify]] | [[Profile__Mem0]] | [[Profile__Volt]] |
|---|---|---|---|---|---|
| **Substrate** | Virtual filesystem (`viking://`) over RAGFS + vector index | Postgres + git-backed memory mode renders blocks as files | Knowledge graph (entity nodes + typed edges) | Vector store + entity store + SQLite | Postgres DAG (immutable + summary) |
| **Unification claim** | Memory + resource + skill in one tree | Memory blocks + archival memory | Memory + sources in one graph | Memory only; resources external | Coding sessions only |
| **Tiering** | L0/L1/L2 per node (~100t / ~2k / full) | Block-level (small in-context, archival) | Graph node properties | None (single payload) | Bindle + summary node |
| **Retrieval** | **Recursive directory walk** + rerank | Block recall + archival vector search | Graph traversal + cypher | Hybrid: semantic + BM25 + entity boost | Bindle expansion + `lcm_grep` |
| **Write policy** | Async L0/L1 generation; 8-category extract loop with merge ops | Block update via LLM tool call | Edge creation/update | Single-pass additive | Verbatim append + summary |
| **Eviction** | Archive + hotness + per-category lifecycle | Block summarization | Edge weight decay | None (OSS) | Dolt evicts bindles |
| **Supersession** | Per-category merge ops with `memory_diff.json` audit | Block-level overwrite | First-class via edges | Not modeled in OSS | Immutable log + summary |
| **Multi-tenancy** | account/user/agent triple, configurable agent isolation, role-based | Org/user/agent (Letta Cloud) | Workspace | user/agent/run required | Conversation-scoped |
| **Transactional model** | Path locks (EXACT/TREE) + redo log | Postgres ACID | Graph DB ACID | None (OSS) | Postgres ACID |
| **CLI / verb surface** | `ov ls`, `tree`, `find`, `grep`, `read`, `abstract`, `overview`, `add-resource` | API + ADE UI | API + UI | `add`/`search`/`update`/`delete` | Terminal-native (`lcm_*`) |
| **Languages** | Rust (FS+CLI) + Python (service) + TS (plugins) + C++ (indices) | Python | Python | Python + TS | TS |
| **License** | AGPL-3.0 (main) + Apache-2.0 (CLI/examples) | Apache-2.0 | Apache-2.0 | Apache-2.0 | Apache-2.0 |
| **Backer / scale** | Volcengine / ByteDance (production) | Letta Inc. | Independent | Mem0.ai | Independent |
| **Best fit** | Multi-agent, multi-user, large-corpus, ops-disciplined deployments | Conversational agents with editable persona/state | Knowledge-graph-shaped domains | General-purpose memory layer | Long-horizon coding sessions |

The closest *architectural* sibling is [[Profile__Letta]] (the only other system that treats memory as files), but Letta's filesystem is a thin git layer over a Postgres-backed block store, while OpenViking's filesystem *is* the substrate. The closest *unification* sibling is [[Profile__Graphify]] (also collapses resource and memory into one store), but Graphify uses a graph and OpenViking uses a tree. Neither comparator combines the bet the way OpenViking does.

## One-line summary

> OpenViking is a Rust-cored virtual filesystem with three abstraction tiers per node, unified `viking://` URIs that collapse memory/resource/skill into one tree, recursive directory-walking retrieval that exploits that tree for both speed and observability, a careful path-lock + redo-log consistency model, and Volcengine-grade production deployment story — the strongest "context-as-filesystem" bet in the study, and the right default whenever path-determinism, hierarchical structure, or large-corpus token economics matter more than the simplicity of a flat key-value memory.
