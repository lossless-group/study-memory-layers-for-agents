# Memory Layers for Agents — Study

## The question

> How do production memory systems for AI agents structure recall —
> across vector, graph, and key-value stores; across user, session, and
> agent scopes; across short-term context and long-term knowledge —
> and which conventions are converging?

The interesting bits are not the embeddings themselves; those are commodity.
The interesting bits are: how does each system *decide* what to remember,
*when* to update, *how* to scope, and *what* shape the memory artifacts
take on disk and on the wire.

## What we are looking at, repo by repo

When reading each entry below, the working checklist is:

- **Storage topology.** Single store or hybrid (vector + graph + KV)? What
  determines which store handles which class of memory?
- **Write policy.** Append-only? Self-editing? LLM-summarized? Reconciled
  against a graph? When does a memory get superseded vs. merged?
- **Scopes & namespacing.** User / session / agent / tenant — how are these
  modeled, and how does retrieval respect them?
- **Schema of a memory.** What fields does a single memory record carry?
  Timestamps, provenance, confidence, embedding ref, references to other
  memories?
- **Recall surface.** Vector similarity, graph traversal, time-based, hybrid
  ranking? What's the API the agent actually calls?
- **Eviction & compaction.** Is there a policy? A summarizer? A TTL?
- **Serialization on disk.** What does the persisted form look like? JSON
  blobs, parquet, a specific graph format, SQL rows? Could a non-AI program
  parse it cleanly?
- **Operational story.** Local-first? Server-required? Stateless agent +
  external store?

---

## Pins last refreshed 2026-11-23

Every submodule now points at the latest commit of its upstream default
branch. Each profile in [`context-v/profiles/`](./context-v/profiles) has a
"Changes since the previous pin" section with the details. Commit counts
include merges. "Latest release" is the newest release at or before the new
pin; where upstream tags nothing, the package version is given.

| Repo | Old → new commit date | Commits | Latest release |
|---|---|---|---|
| beads | 2026-05-27 → 2026-11-20 | 2,541 | v1.3.4 (v1.4.0-rc.4 on a release branch) |
| byterover-cli | 2026-05-25 → 2026-06-26 | 204 | 3.16.1 (npm; no version tags) |
| delta-mem | 2026-05-15 → 2026-09-25 | 6 | none tagged |
| graphify | 2026-05-20 → 2026-11-23 | 2,666 | 0.9.115 (PyPI) |
| graphiti | 2026-05-14 → 2026-11-22 | 256 | graphiti-core v0.31.2, mcp-v1.1.3 |
| hindsight | 2026-05-26 → 2026-11-21 | 2,768 | v1.2.3 (1.0 shipped 2026-10-13) |
| honcho | 2026-05-21 → 2026-11-21 | 432 | v3.6.2 |
| letta | 2026-05-14 → 2026-10-18 | 12 | 0.16.8 (server code archived) |
| mem0 | 2026-04-29 → 2026-11-18 | 515 | Python v2.2.1, ts-v3.3.1 |
| mempalace | 2026-05-17 → 2026-11-22 | 1,662 | v3.12.0 |
| neo | 2026-05-02 → 2026-11-08 | 363 | 0.63.0 (final; repo retired) |
| openviking | 2026-05-26 → 2026-11-23 | 2,305 | v0.5.5 |
| paxm | 2026-07-20 → 2026-08-28 | 20 | v0.2.5 |
| retaindb | 2026-05-24 → 2026-06-12 | 8 | none tagged (@retaindb/local 0.2.2) |
| statebench | 2026-03-27 → 2026-10-07 | 72 | v2.0.0 |
| supermemory | 2026-05-26 → 2026-11-22 | 372 | server-v0.0.9 (SDKs untagged) |
| understand-anything | 2026-06-19 → 2026-11-22 | 305 | v2.9.8 |
| volt | 2026-02-27 → 2026-02-27 | 0 | v0.6.2 (unchanged) |

Status changes a reader should know about:

- **Graphify** relicensed from MIT to Apache-2.0 (2026-07-22). The README
  now names `Graphify-Labs/graphify` as the official repo.
- **Letta**'s `main` became a landing page on 2026-08-15. The V1 server
  code profiled here moved to the `archive` branch, and active work is in
  `letta-ai/letta-code`.
- **Neo** was retired on 2026-11-08; its README says development continues
  in CAR. 0.63.0 is the last release.
- **StateBench** 2.0 found defects in its own scorer and retracted the 1.x
  numbers.

---

## The design space at a glance

| Bet | Entry |
|---|---|
| Memory baked into the model (frozen-backbone transformer adapter) | [Delta-Mem](./delta-mem) |
| Real typed bi-temporal graph | [Graphiti](./graphiti) |
| Self-editing core memory + paged archival (the MemGPT lineage; V1 server now archived) | [Letta](./letta) |
| Hybrid vector + entity-link + tight CRUD API | [Mem0](./mem0) |
| Verbatim chunks + hybrid recall, no extraction | [MemPalace](./mempalace) |
| Typed/scoped facts + deterministic supersession + outcome learning (retired 2026-11) | [Neo](./neo) |
| Immutable Postgres log + summary DAG | [Volt](./volt) |
| Folder → in-process NetworkX KG + Leiden + MCP query surface | [Graphify](./graphify) |
| Multi-agent LLM → one knowledge-graph.json, Zod-normalized; "graphs that teach" (grep + Astro dashboard, plus a small read-only MCP server since 2.9) | [Understand-Anything](./understand-anything) |
| Multi-peer "theory of mind" — what A knows about B, derived async | [Honcho](./honcho) |
| Context as a filesystem; L0/L1/L2 abstract tiers; unified memories + resources + skills; subtree-scoped vector search | [OpenViking](./openviking) |
| Typed memory taxonomy (world facts / experiences / mental models) + bank-based scoping | [Hindsight](./hindsight) |
| 12+ semantic categories with bi-temporal validity (validFrom/validUntil) | [RetainDB](./retaindb) |
| Universal RAG + memory layer; vendor-reported top scores on LongMemEval / LoCoMo / ConvoMem; engine not in the repo | [Supermemory](./supermemory) |
| Coding-agent context tree (CLI + optional cloud sync; Elastic-licensed; the calling agent does the LLM work since mid-2026) | [ByteRover CLI](./byterover-cli) |
| Procedural/working memory — versioned dependency-typed task graph on a Git-like SQL DB (Dolt), no vectors | [Beads](./beads) |
| Local-first SQLite/FTS5 default with pluggable remote backends (Zep/Mem0/MemOS/OpenViking); durable local write-queue decouples agent turns from provider latency | [Paxm](./paxm) |
| Conformance benchmark (the yardstick; v2.0 re-scored after auditing its own scorer) | [StateBench](./statebench) |

Deep per-entry write-ups live in [`context-v/profiles/`](./context-v/profiles).

## In the study now

### [mem0](./mem0)
- **Repo:** https://github.com/mem0ai/mem0 — *Universal memory layer for AI Agents*
- **Maintainer:** Mem0.ai (`mem0ai` org)
- **Why this is here:** Pioneers a composable hybrid architecture (vector +
  graph + KV store) with adaptive updates. Multi-level recall across
  user / session / agent scopes is its headline. Reported +26% accuracy
  over OpenAI memory and 91% faster responses in their benchmarks. Most
  starred entry in this space (54k+ at the original pin) and the most
  explicit about being a *layer* rather than a framework. Since the May
  2026 pin, the docs say graph memory is a Platform feature removed from
  OSS, `add()`/`update()` take an `expiration_date` that hides (never
  deletes) expired memories, and the README's LoCoMo / LongMemEval scores
  (vendor-reported) are attributed to the managed platform rather than
  the open-source SDK.

### [neo](./neo)
- **Repo:** https://github.com/Parslee-ai/neo — *A self-improving code
  reasoning engine with persistent semantic memory*
- **Maintainer:** Parslee AI (`Parslee-ai` org)
- **Why this is here:** A reasoning engine, not a framework — the memory
  is *the point*, not a feature bolted on. Worth comparing schema and
  write-policy choices against Mem0 and Graphiti. Smaller surface than
  the major players, which makes it easier to read end-to-end.
- **Status:** retired on 2026-11-08. The README says the repo is archived
  and development continues in CAR; 0.63.0 is the final release. The pin
  stays as a finished reference design.

### [delta-mem](./delta-mem)
- **Repo:** https://github.com/declare-lab/delta-Mem —
  *δ-mem: Efficient Online Memory for Large Language Models*
- **Maintainer:** declare-lab (SUTD) — Jingdi Lei, Di Zhang, Junxian Li,
  Weida Wang, Kaixuan Fan, Xiang Liu, Qihan Liu, Xiaoteng Ma, Baian Chen,
  Soujanya Poria
- **Why this is here:** The architectural counter-bet to every other
  entry. While Mem0, MemPalace, Graphiti, Neo, and Volt all answer "where
  does the agent store and look things up," Delta-Mem answers "what if
  memory was part of the model's forward pass?" A frozen-backbone
  transformer adapter that gives each attention head a low-rank dense
  **state matrix** updated by a learned **delta rule** (S_{t+1} = λ·S_t
  − β·(S_t·k_t)⊗k_t + β·v_t⊗k_t), with three temporal write
  granularities (TSW/SSW/MSW), a Triton-accelerated affine scan, and a
  public Qwen3-4B-Instruct adapter on Hugging Face. Released alongside
  [arXiv:2605.12357](https://arxiv.org/abs/2605.12357). Evaluated on
  LoCoMo, HotpotQA, IFEval, GPQA Diamond, and MemoryAgentBench. This is
  a research artifact (not a deployable library), and that's the point —
  it forces the question "is agent memory even a retrieval problem?"
  that the system-level entries quietly assume. Upstream has changed
  little since the May 2026 pin: the README now titles it a NeurIPS 2026
  paper, and the experimental `synthetic_kv`, `latent_context`, and
  `memory_branch` readouts were removed.

### [letta](./letta)
- **Repo:** https://github.com/letta-ai/letta — *Platform for building
  stateful agents: AI with advanced memory that can learn and
  self-improve over time*
- **Maintainer:** letta-ai (formerly MemGPT-ai; Charles Packer, Sarah
  Wooders et al.)
- **Why this is here:** The direct successor to **MemGPT** — the 2023
  Berkeley paper that named the agent-memory problem and shipped the
  OS-inspired hierarchical-memory pattern (core context = RAM, archival
  = disk, recall = paging). The MemGPT lineage is the framing fact.
  Letta is the operational productization: every agent is a persistent
  Postgres row, **core memory** is a set of agent-editable **Blocks**
  rendered into the system prompt, **archival memory** is a paginated
  pgvector-backed passage store with semantic search. The agent edits
  its own memory via `core_memory_append` / `core_memory_replace` —
  the headline MemGPT design move that no other entry in the study
  replicates. Ships as a FastAPI + Postgres + pgvector compose stack
  with REST, WebSocket, and an **OpenAI-compatible
  `/v1/chat/completions` endpoint** so a Letta agent looks like a
  ChatGPT-shaped model to any client. **Multi-agent block sharing** is
  a database join (a single Block row can belong to many agents). The
  newer **git-backed memory mode** renders blocks as files
  (`system/persona.md`, `skills/.../SKILL.md`) and turns every memory
  edit into a commit — making it the closest entry in the study to our
  own `context-vigilance` discipline, but with the agent as committer.
- **Status:** on 2026-08-15 upstream turned `main` into a landing page and
  moved the V1 server code to the `archive` branch; active development is
  in `letta-ai/letta-code`. The pinned `main` commit no longer contains the
  server source, so the profile's paths resolve on `origin/archive`.

### [graphiti](./graphiti)
- **Repo:** https://github.com/getzep/graphiti — *Build Real-Time Knowledge
  Graphs for AI Agents*
- **Maintainer:** Zep Software, Inc. (Paul Paliychuk, Preston Rasmussen,
  Daniel Chalef)
- **Why this is here:** The only entry in the study that puts a real,
  typed, **bi-temporal graph** between the agent and its memories — four
  interchangeable backends (Neo4j, FalkorDB, Neptune, and the embedded
  Relate driver that replaced Kuzu in late 2026), a single-LLM-call episode-to-graph extraction pipeline with
  Pydantic-typed entities, hybrid BM25 + cosine + BFS recall with
  cross-encoder reranking, and label-propagation community detection.
  Every edge carries both `valid_at`/`invalid_at` (real-world validity)
  and `created_at`/`expired_at` (system time) — textbook bi-temporality
  brought to agent memory. Backed by a peer-reviewed paper
  ([*Zep: A Temporal Knowledge Graph Architecture for Agent
  Memory*](https://arxiv.org/abs/2501.13956)). The interesting
  comparison is Graphiti vs MemPalace: opposite bets on structure
  (MemPalace says structure is over-engineered; Graphiti says it's
  under-engineered).

### [mempalace](./mempalace)
- **Repo:** https://github.com/MemPalace/mempalace — *The best-benchmarked
  open-source AI memory system. And it's free.*
- **Maintainer:** MemPalace Contributors (`MemPalace` org; milla-jovovich,
  @bensig)
- **Why this is here:** The most direct counter-bet to Mem0 in the study.
  Same problem space (give agents persistent memory), nearly opposite
  design choice at the write step: MemPalace stores **verbatim text** —
  no LLM-driven extraction, no summarization — in ChromaDB
  (`mempalace_drawers` + `mempalace_closets`) plus a local SQLite
  temporal entity graph. Recall is hybrid (semantic + BM25 + closet-boost,
  closets *signal* never *gate*), and the published benchmarks
  (`benchmarks/BENCHMARKS.md`) report 96.6% R@5 on LongMemEval with **zero
  LLM calls** at query time, and 92.9% vs Mem0's 30–45% on ConvoMem — a
  ~2× margin attributed directly to extraction losing information.
  Inspired by Zettelkasten + Method of Loci (`MISSION.md`). The
  benchmark-honesty note in `BENCHMARKS.md:70-95` (the 100% headline
  involved teaching-to-test; the held-out figure is 98.4%) is worth
  reading in its own right. Since the May 2026 pin (v3.3.5 → v3.12.0):
  re-mines journal overwritten drawer text to `drawer_revisions.jsonl`
  instead of losing it, five opt-in vector backends sit beside Chroma
  (`sqlite_exact`, `rust_exact`, `milvus`, `qdrant`, `pgvector`), the
  side knowledge graph gained `kg_supersede`, and the MCP server grew from
  29 to 47 tools.

### [volt](./volt)
- **Repo:** https://github.com/Martian-Engineering/volt — *Coding agent with
  lossless context management*
- **Maintainer:** Martian Engineering (`Martian-Engineering` org)
- **Why this is here:** A coding agent built around **Lossless Context
  Management (LCM)** — a dual-state design where every user message,
  assistant response, and tool result is persisted verbatim (immutable
  store) and the active context is assembled from recent raw messages plus
  precomputed summary nodes. Storage is a DAG in embedded Postgres
  (`voltcode_lcm`, optional external via `LCM_DATABASE_URL`). The
  write/summarize policy is **deterministic** (soft/hard token thresholds
  drive a control loop), not LLM-decided — a useful contrast against
  Mem0's adaptive updates and Letta's self-editing memory. Two runtime
  modes (Dolt: evict oldest with ghost-cue off-context retrieval; Upward:
  recursive bottom-up condensation, default) make the eviction/compaction
  trade-off explicit and readable.

### [graphify](./graphify)
- **Repo:** https://github.com/safishamsi/graphify — *AI coding assistant
  skill that turns any folder of code, docs, papers, images, or videos
  into a queryable knowledge graph*
- **Maintainer:** Safi Shamsi (`safishamsi`; the README now names
  `Graphify-Labs/graphify` as the official repo); Apache-2.0 since
  2026-07-22 (MIT before); PyPI package `graphifyy` (0.9.115 at this pin)
- **Why this is here:** The codebase-as-memory bet. Where every other
  entry models *conversational* memory (user / session / agent turns),
  Graphify models the **static corpus an agent works against** as a
  persistent, queryable knowledge graph and exposes it to the agent via
  **MCP** (`graphify/serve.py:1`). The pipeline is single-process and
  legible end-to-end: `detect → extract → build → cluster → analyze →
  report → export` (`ARCHITECTURE.md`), with tree-sitter parsers for
  ~40 languages (`graphify/extract.py` plus `graphify/extractors/`)
  emitting `{nodes, edges}` dicts that get folded into a
  NetworkX graph, **Leiden community detection** via graspologic with a
  NetworkX-Louvain fallback (`graphify/cluster.py:122-282`), and
  per-extraction **confidence labels** (`EXTRACTED` / `INFERRED` /
  `AMBIGUOUS`) carried on every edge so the agent can reason about how
  much to trust each relation. The MCP server exposes graph-query tools
  (`query_graph`, `get_neighbors`, `get_community`, `god_nodes`,
  `shortest_path`, PR-triage tools) — so "memory" here is structural
  recall over the project, not episodic recall over a conversation. The
  interesting contrast is against Graphiti and MemPalace: Graphiti puts
  a *bi-temporal* graph DB between agent and memory; MemPalace stores
  verbatim chunks; Graphify denies the graph-DB premise altogether and
  ships the graph as a single `graph.json` next to a static HTML viewer.
  A useful "do we even need a server?" data point in a study otherwise
  dominated by server-backed designs. New since the May 2026 pin: 0.9.0
  switched node IDs to full repo-relative paths (a breaking change for
  saved graphs), the MCP server can run over HTTP as well as stdio, and
  `graphify save-result` / `graphify reflect` write a `LESSONS.md` without
  an LLM call, the first feature that records what the agent learned
  rather than what the corpus contains.

### [understand-anything](./understand-anything)
- **Repo:** https://github.com/Egonex-AI/Understand-Anything — *Turn any
  codebase, knowledge base, or docs into an interactive knowledge graph you
  can explore, search, and ask questions about*
- **Maintainer:** Egonex-AI (`Egonex-AI` org; originally Lum1104 / Yuxiang
  Lin, © Infinite Universe, Inc.); MIT; TypeScript plugin v2.9.8
- **Why this is here:** The study's matched pair to **Graphify** — same
  *codebase-as-memory* bet (model the static corpus, not conversation, as a
  knowledge graph), same tree-sitter substrate, same MIT multi-platform-plugin
  packaging — executing it **oppositely on three axes**, which is exactly what
  makes the pairing worth reading. (1) **Extraction:** Graphify is
  deterministic (tree-sitter emits canonical types); Understand-Anything runs a
  **multi-agent LLM pipeline** (nine sub-agents — `project-scanner` →
  `file-analyzer` → `architecture-analyzer` → `domain-analyzer` →
  `article-analyzer` → `tour-builder` → `graph-reviewer` →
  `assemble-reviewer`) and then **repairs the LLM's output** against a Zod
  schema with explicit alias maps (`fn`/`method` → `function`, `extends` →
  `inherits`, `conflicts_with` → `contradicts`) — the cleanest worked example
  in the study of *how you trust a graph an LLM authored*
  (`understand-anything-plugin/packages/core/src/schema.ts:18-76`). (2) **Recall:** Graphify ships an MCP
  query server; Understand-Anything's skills (`/understand`, `/understand-chat`,
  `/understand-diff`, …) instruct the agent to **`grep` the single
  `.ua/knowledge-graph.json`** (`.understand-anything/` before 2.9, still honored
  when present), and an Astro **dashboard** + dependency-ordered **guided tours**
  serve humans. Since the June 2026 pin it also ships an optional read-only stdio
  MCP server with six tools (status / search / explain / diff / onboard /
  dashboard) that return explanations and impact reports rather than raw graph
  traversal.
  Its ethos is "**graphs that teach > graphs that impress**." (3) **Scope:** a
  `kind: "codebase" | "knowledge"` switch plus five **knowledge** node types
  (article / entity / topic / claim / source) and knowledge edges
  (`cites` / `contradicts` / `builds_on`) point the same machinery at *prose and
  argument* (Karpathy-style LLM wikis, papers) — the only entry in the study
  that models a knowledge base as a first-class graph subject. A third kind,
  `"design"`, now covers Figma files. The schema is
  rich (27 node types, 38 edge types across 9 categories; `types.ts:1-20`) and
  the 42 tree-sitter language configs reach well past code into infra and docs
  (`terraform`, `kubernetes`, `dockerfile`, `openapi`, `markdown`). Deep
  write-up:
  [`context-v/profiles/Profile__Understand-Anything.md`](./context-v/profiles/Profile__Understand-Anything.md).

### [honcho](./honcho)
- **Repo:** https://github.com/plastic-labs/honcho — *Build AI agents
  that truly know your users*
- **Maintainer:** Plastic Labs (`plastic-labs` org); AGPL-3.0
- **Why this is here:** The only entry in the study that puts a
  **multi-peer / theory-of-mind** model at the center of the design.
  Where Mem0 models memories *belonging to a user* and Letta models
  memories *editable by an agent*, Honcho models *what one peer
  understands about another* — `peers`, `sessions`, `messages`,
  `workspaces` as primary entities, with vector-embedded "internal
  collections" keyed by the (observer, observed) pair. Write path is
  **append-only messages + async background reasoning**: new messages
  immutably persist, and a *deriver* worker generates summaries, peer
  cards, and conclusions out-of-band. This is closer to the "agents
  building rich models of users" framing that Plastic Labs' research
  thread (Open Source Honcho, the "Tutor-GPT" lineage) has been chasing
  since 2023, and it's the cleanest example in the study of separating
  *storage* from *understanding*. Worth comparing the deriver to Volt's
  deterministic summary thresholds — both are explicit, neither blocks
  the agent loop. Since the May 2026 pin (v3.0.7 → v3.6.2): 3.6.0 added
  multi-tenancy, so every table carries `tenant_id` under row-level
  security and collections are now unique on (tenant, observer, observed,
  workspace); conclusions cite their source messages; and Qdrant and
  ChromaDB joined the vector backends.

### [openviking](./openviking)
- **Repo:** https://github.com/volcengine/OpenViking — *An open-source
  context database for AI agents (file-system paradigm)*
- **Maintainer:** Volcengine (ByteDance's cloud division); AGPL-3.0
  main / Apache-2.0 CLI + examples
- **Why this is here:** The most architecturally distinct entry on the
  storage side. OpenViking refuses the flat-vector premise entirely and
  organizes context as a **filesystem** — `viking://resources/`,
  `viking://user/`, `viking://agent/` — with three abstraction tiers
  per node (**L0** one-sentence abstract, **L1** overview, **L2** full
  data) so retrieval can match at the coarsest tier and drill down
  on demand. Retrieval used to be a recursive directory walk; since
  v0.4.23 (2026-09) it is one vector search scoped to a subtree, then one
  rerank pass. Unifies what every other study entry
  treats as separate concerns: **memories** (user / agent task),
  **resources** (docs, repos), and **skills** (agent instructions)
  are all just nodes in the same tree. ByteDance-scale backing is the
  operational story (Rust core in `crates/`, multi-language CLI, Docker
  deploy). The interesting compare-and-contrast is against Letta's
  git-backed memory mode (which also renders memory as files but on a
  flat-ish layout) and against Graphify (which also collapses
  resource-as-context but as a knowledge graph rather than a tree). Other
  changes by v0.5.5: the Python embedded mode is gone (server only), the
  agent identity dimension was replaced by account / user / peer, and a
  new OpenViking Gateway proxies an agent's model API and adds memory to
  each turn.

### [hindsight](./hindsight)
- **Repo:** https://github.com/vectorize-io/hindsight — *Agent memory
  system that helps AI agents learn over time*
- **Maintainer:** Vectorize.io (`vectorize-io` org); MIT
- **Why this is here:** The first entry in the study to commit to a
  **typed memory taxonomy** as a first-class axis: every memory is one
  of *world fact*, *experience*, or *mental model*. Storage is
  deliberately polyglot — vector for semantics, graph for entity /
  causal links, time series for temporal context, BM25 for lexical
  match — and recall is a three-verb surface (`retain`, `recall`,
  `reflect`) where `reflect` is the unusual one: the agent analyzes
  its *own existing memories* rather than the world. Per-user /
  per-agent isolation lives in a "bank" abstraction. Ships as a full
  open-source stack — Python API, Node/Python SDKs, CLI, Docker /
  Helm — not a wrapper around a hosted service. The compare-and-
  contrast against Mem0 is direct: same hybrid-retrieval thesis, but
  Hindsight argues the *type of memory* (epistemic vs. experiential
  vs. inferential) should drive write, not just retrieval ranking.
  Hindsight reached 1.0 on 2026-10-13 and v1.2.3 at this pin. The
  per-agent plugins (Claude Code, Codex, opencode, Cline, and others)
  were replaced by one cross-agent package,
  `hindsight-integrations/coding-agents/`
  (`@vectorize-io/hindsight-coding-agents`), which keeps one memory bank
  per repository shared by every agent working it and seeds that bank
  from git history and past sessions. The Claude Code marketplace at the
  repo root was deleted on 2026-10-23.

### [retaindb](./retaindb)
- **Repo:** https://github.com/RetainDB/RetainDB — *Durable memory for
  AI agents — decisions, preferences, workflows, corrections, project
  facts, session handoffs that survive across conversations*
- **Maintainer:** RetainDB organization; Apache-2.0 (local / SDK / MCP),
  BSL-1.1 (server)
- **Why this is here:** Pushes the typed-memory bet further than
  Hindsight — **12+ semantic categories** (factual, preference,
  procedural, decision, constraint, correction, session-summary, etc.)
  — and pairs it with **bi-temporal validity** (`validFrom` /
  `validUntil`) so superseding stale facts is an explicit, queryable
  operation rather than something the recall ranker has to paper over.
  Hybrid retrieval is "lexical + vector + graph signals + RRF fusion +
  reranking." Three deployment modes (Local single-machine, Server with
  Postgres + pgvector, managed Cloud) cover the same "library / server
  / hosted" trifecta Mem0 ships, but the BSL-licensed server is the
  flag for anyone planning on commercial hosting. The contrast against
  Mem0 is direct on the supersession axis (Mem0 has none in OSS;
  RetainDB makes it a first-class column) and against Neo on the
  *typed-categories-vs-typed-facts* axis (Neo carries a `superseded_by`
  chain on a single fact type; RetainDB enumerates categories). Since the
  May 2026 pin the local runtime gained an agent filesystem
  (`.retaindb/files/`) and connectors; the `LICENSE-BSL` file the README
  references is still absent.

### [supermemory](./supermemory)
- **Repo:** https://github.com/supermemoryai/supermemory — *The memory
  and context layer for AI*
- **Maintainer:** supermemoryai research lab; MIT
- **Why this is here:** The vendor-reported benchmark-leaderboard occupant —
  self-reported **#1 on LongMemEval, LoCoMo, and ConvoMem** as of mid-2026,
  the three headline harnesses every other entry in this study reports
  against (compare to MemPalace's 96.6% R@5 on LongMemEval and Mem0's
  91.6 / 93.4 on LoCoMo / LongMemEval — the leaderboard moves fast).
  At ~22.7k stars and 1,600+ commits at the original pin (not
  re-verified), one of the most active OSS entries. Design surface is the **universal RAG + memory** bet:
  fact extraction from conversations + user-profile maintenance +
  hybrid search + multimodal ingest + external connectors (Google
  Drive, Notion, GitHub) that sync content into the same store. Scopes
  are `containerTag` (typically a user id) plus optional project tags.
  Claims to handle "temporal changes, contradictions, and automatic
  forgetting" — which makes the comparison against StateBench (the
  yardstick) the obvious next step. Since the May 2026 pin: a breaking v5
  API renames `containerTag` to `namespace`, the MCP server was rewritten
  under `apps/mcp/src/server/`, and a self-hosted engine binary
  (`supermemory-server`) exists, but the docs say it is built from a
  non-public codebase, so the engine is still not in this repo.

### [byterover-cli](./byterover-cli)
- **Repo:** https://github.com/campfirein/byterover-cli — *Memory Hub
  for AI coding agents to remember*
- **Maintainer:** ByteRover team (`campfirein` org)
- **License note:** **Elastic License 2.0** — source-available, not
  OSI-open-source. Commercial use is restricted. We pin it for study
  reading; do not vendor the code into a product without reading the
  license.
- **Why this is here:** The only entry in the study focused
  specifically on **coding-agent memory** as an end-user product
  (compare to Volt, which is also coding-agent-shaped but ships as a
  full agent; ByteRover is *just* the memory hub). Organizes codebase
  knowledge into a **context tree** with local file-based storage and
  optional cloud sync. Project-level scope (per directory),
  multi-machine access, team sharing via ByteRover Cloud. Integrates
  with Cursor, Claude Code, and other coding agents as the consumer
  surface. Since mid-2026 ByteRover no longer runs its own LLM for
  curate, query, or dream: "tool mode" is the default and the calling
  agent does that work. Topic files moved from Markdown to HTML with
  `<bv-*>` elements (`brv migrate` converts a tree), and the
  `providers` / `model` commands were removed. The reason it earns a pin despite the licensing caveat:
  the **CLI + REPL + dashboard** packaging is a distinct deployment
  story (most other entries assume the agent is the only consumer; here
  the human and the agent are both editors of the same context tree)
  and worth studying as one possible answer to "how does a *team* of
  developers share agent memory."

### [beads](./beads)
- **Repo:** https://github.com/gastownhall/beads — *Distributed graph
  issue tracker for AI agents, powered by Dolt*
- **Maintainer:** `gastownhall` org; Go module path
  `github.com/steveyegge/beads` (Steve Yegge et al.); MIT. Ships as
  `@beads/bd` (npm), `beads-mcp` (PyPI), `beads` (Homebrew).
- **Why this is here:** The entry that names the missing axis. Every other
  entry models *episodic / semantic* memory — recall over conversation or
  facts. Beads models **procedural / working memory**: the agent's own
  task graph, so a coding agent can survive context compaction, session
  resets, and account rotations without losing the plot or redoing work.
  And it makes the study's sharpest storage bet — **no vector store, no
  embeddings, no semantic similarity anywhere.** Recall is SQL views +
  graph traversal over a typed dependency graph (`blocks`, `parent-child`,
  `supersedes`, `relates-to`, `discovered-from`, ~19 edge kinds) stored in
  **[Dolt](https://github.com/dolthub/dolt)** — a Git-like, versioned,
  MySQL-compatible database with cell-level merge and native branching.
  **Hash-based IDs** (`bd-a1b2` = base36 of a SHA256 over
  title+description+creator+timestamp+nonce, `internal/idgen/hash.go`) give
  conflict-free writes across parallel agents and branches — no shared
  auto-increment counter to collide on. The write path is mutate-the-row +
  append to an immutable `events` audit trail; supersession is an explicit
  `supersedes` graph edge (compare Neo / RetainDB). Eviction is real:
  closed issues get **Claude-Haiku-summarized "memory decay"**
  (`internal/compact/`) into Summary / Key Decisions / Resolution, with the
  full original preserved in snapshot tables for recovery. A separate,
  deliberately dumb memory surface — `bd remember` / `recall` / `memories`
  / `forget` (`cmd/bd/memory.go`) — stores slugified key→value insights in
  the config KV table and re-injects them every session via `bd prime`
  (`cmd/bd/prime.go`) as a SessionStart hook. On disk, the Dolt DB under
  `.beads/` is authoritative; `.beads/issues.jsonl` is a plain-JSON export
  for viewing/interchange only — *not* the source of truth. Local-first and
  git-optional (`BEADS_DIR` + `--stealth`), with peer-to-peer **federation**
  carrying data-sovereignty tiers (the only GDPR-tier knob in the study).
  The cleanest pairing is Beads vs Volt (both coding-agent, both
  database-backed — and mind the trap: Volt's *eviction mode* is named
  "Dolt" while Beads is *built on* Dolt the DB); the cleanest contrast is
  Beads vs every vector-backed entry. Since the May 2026 pin (v1.0.4 →
  v1.3.4 stable): local storage is still Dolt-only (PostgreSQL and MySQL
  adapters were added and rolled back before release); new are `bd serve`
  (HTTP), an experimental HTTP backend (`bd connect`), an opt-in events
  journal, and size caps on what `bd prime` injects. Recall still uses no
  embeddings. Deep write-up:
  [`context-v/profiles/Profile__Beads.md`](./context-v/profiles/Profile__Beads.md).

### [paxm](./paxm)
- **Repo:** https://github.com/pax-beehive/paxm — *Persistent, provider-neutral
  memory for Codex, Claude Code, OpenCode, Pi, and MCP coding agents*
- **Maintainer:** `pax-beehive` org; Apache-2.0
- **Why this is here:** The routing/adapter bet, not a storage-architecture
  bet. Default backend is local SQLite + FTS5/BM25 for turn-level memories —
  zero account, API key, embeddings, or extra model calls to get started —
  but it's explicitly a thin layer in front of swappable remote providers
  (Zep, Mem0, MemOS, OpenViking, or a custom JSON-RPC adapter), with
  multiple provider instances routable simultaneously via profiles. That
  makes it the study's clearest example of "memory as a pluggable backend
  behind one stable agent-facing API" rather than "memory as a bespoke
  store." Write path is a durable local queue with background,
  resumable delivery to the provider — hook acknowledgement only waits on
  the local transaction, so a slow or down remote provider never blocks
  the agent turn. Records carry origin (user/agent/session/turn) and scope
  (personal/team) metadata that providers supporting attribution must
  round-trip. Recall surface is three-way: CLI (`paxm recall --query`),
  four MCP tools (`paxm_recall`, `paxm_remember`, `paxm_history`, +1), and
  passive injection via lifecycle hooks under an 800ms budget. Integration
  breadth is the other headline — native Claude Code plugin (5 hooks),
  native Codex plugin, local OpenCode/Pi plugins, plus a stdio MCP server —
  more agent-harness surface area than most other entries in this study
  attempt. v0.2.5 at this pin adds a `team-memory` backend that sends
  each write under its own agent's credential from a local paxl device,
  and an optional per-provider cap on recall hits. Early-stage (152★,
  34 releases, 161 commits at the original pin) with active
  CI/CD including optional real-agent regression evals. Deep write-up:
  [`context-v/profiles/Profile__Paxm.md`](./context-v/profiles/Profile__Paxm.md) —
  covers the squared-reciprocal-rank cross-provider ranking calibrator
  (`internal/memory/ranking.go`), the checksummed SQLite-WAL durable write
  queue (`internal/capturequeue/queue.go`), and the exact-fingerprint (not
  semantic) LTM admission policy.

### [statebench](./statebench)
- **Repo:** https://github.com/Parslee-ai/statebench — *Conformance test
  for stateful AI agents. Measures state correctness over time.*
- **Maintainer:** Parslee AI (`Parslee-ai` org)
- **Why this is here:** A *benchmark* rather than an implementation —
  fills a different slot in the study. Pinning it lets us evaluate the
  memory-layer implementations against a common harness, instead of
  trusting each project's self-reported numbers. Pair it with Mem0 and
  neo as the units under test. v2.0.0 (2026-08-04) audited its own scorer
  and found six defects; the worst counted correct answers that named a
  superseded value in order to reject it as resurrections. The 1.x
  numbers are retracted, so cite only 2.x results.

---

## Candidates to add (verified to exist on GitHub)

These are not yet pinned as submodules. When the study expands, run
`git submodule add <url> <slug>` from the study root to add any of them.

### Memory-specialized

- **Zep / Graphiti** — temporal knowledge graphs for session memory.
  Integrates with LangChain / LangGraph. The interesting code lives in
  Graphiti, not in the Zep examples repo.
  - https://github.com/getzep/graphiti — *Build Real-Time Knowledge Graphs
    for AI Agents* (25k★ — recommended entry point)
  - https://github.com/getzep/zep — examples and integrations (4.5k★)

- **Letta** — open-source server with self-editing memory, descended from
  MemGPT. Stateful agents that persist user preferences and survive
  conversation resets.
  - https://github.com/letta-ai/letta (22k★)

- **LangMem** — LangChain's memory utilities; summarization for context
  limits. Smaller surface than the others but tightly integrated with
  the LangChain stack.
  - https://github.com/langchain-ai/langmem (1.4k★)

- **Memary** — graph-centric memory layer for autonomous agents. Worth
  comparing schema choices against Mem0 and Graphiti.
  - https://github.com/kingjulio8238/Memary (2.6k★)

- **Cognee** — pipelines for RAG-style memory; positions itself as
  "memory for your AI agents in 6 lines of code." Focus on ingestion
  ergonomics.
  - https://github.com/topoteretes/cognee (17k★)

### Broader frameworks (not memory-first, but ship memory primitives)

These are large enough that they will dilute the study if added wholesale.
Worth referencing only if a specific subdir is what we want to study.

- **LangChain** — modular memory buffers and summary memory live somewhere
  in here. https://github.com/langchain-ai/langchain (135k★)
- **LlamaIndex** — document-integrated memory and retrieval.
  https://github.com/run-llama/llama_index (49k★)
- **Cloudflare Agents** — Workers-runtime agent memory and ingestion.
  https://github.com/cloudflare/agents (4.8k★)

---

## Reading order suggestion

1. Start with **Mem0**'s README and `docs/` for the topology overview, then
   walk the source for the write/update path (look for "add", "update",
   "consolidate" verbs in the public API).
2. Compare to **Graphiti**'s temporal-graph approach — same problem,
   different storage primitive.
3. Compare to **Letta**'s self-editing memory — same problem, different
   *control* primitive (the agent edits memory, not the framework).

By the time those three are read in sequence, the design space should be
mostly visible.
