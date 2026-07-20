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

## The design space at a glance

| Bet | Entry |
|---|---|
| Memory baked into the model (frozen-backbone transformer adapter) | [Delta-Mem](./delta-mem) |
| Real typed bi-temporal graph | [Graphiti](./graphiti) |
| Self-editing core memory + paged archival (the MemGPT lineage) | [Letta](./letta) |
| Hybrid vector + entity-link + tight CRUD API | [Mem0](./mem0) |
| Verbatim chunks + hybrid recall, no extraction | [MemPalace](./mempalace) |
| Typed/scoped facts + deterministic supersession + outcome learning | [Neo](./neo) |
| Immutable Postgres log + summary DAG | [Volt](./volt) |
| Folder → in-process NetworkX KG + Leiden + MCP query surface | [Graphify](./graphify) |
| Multi-agent LLM → one knowledge-graph.json, Zod-normalized; "graphs that teach" (grep + Astro dashboard, not a query server) | [Understand-Anything](./understand-anything) |
| Multi-peer "theory of mind" — what A knows about B, derived async | [Honcho](./honcho) |
| Context as a filesystem; L0/L1/L2 abstract tiers; unified memories + resources + skills | [OpenViking](./openviking) |
| Typed memory taxonomy (world facts / experiences / mental models) + bank-based scoping | [Hindsight](./hindsight) |
| 12+ semantic categories with bi-temporal validity (validFrom/validUntil) | [RetainDB](./retaindb) |
| Universal RAG + memory layer, currently top of LongMemEval / LoCoMo / ConvoMem | [Supermemory](./supermemory) |
| Coding-agent context tree (CLI + optional cloud sync; Elastic-licensed) | [ByteRover CLI](./byterover-cli) |
| Procedural/working memory — versioned dependency-typed task graph on a Git-like SQL DB (Dolt), no vectors | [Beads](./beads) |
| Local-first SQLite/FTS5 default with pluggable remote backends (Zep/Mem0/MemOS/OpenViking); durable local write-queue decouples agent turns from provider latency | [Paxm](./paxm) |
| Conformance benchmark (the yardstick) | [StateBench](./statebench) |

Deep per-entry write-ups live in [`context-v/profiles/`](./context-v/profiles).

## In the study now

### [mem0](./mem0)
- **Repo:** https://github.com/mem0ai/mem0 — *Universal memory layer for AI Agents*
- **Maintainer:** Mem0.ai (`mem0ai` org)
- **Why this is here:** Pioneers a composable hybrid architecture (vector +
  graph + KV store) with adaptive updates. Multi-level recall across
  user / session / agent scopes is its headline. Reported +26% accuracy
  over OpenAI memory and 91% faster responses in their benchmarks. Most
  starred entry in this space (54k+) and the most explicit about being a
  *layer* rather than a framework.

### [neo](./neo)
- **Repo:** https://github.com/Parslee-ai/neo — *A self-improving code
  reasoning engine with persistent semantic memory*
- **Maintainer:** Parslee AI (`Parslee-ai` org)
- **Why this is here:** A reasoning engine, not a framework — the memory
  is *the point*, not a feature bolted on. Worth comparing schema and
  write-policy choices against Mem0 and Graphiti. Smaller surface than
  the major players, which makes it easier to read end-to-end.

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
  that the system-level entries quietly assume.

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

### [graphiti](./graphiti)
- **Repo:** https://github.com/getzep/graphiti — *Build Real-Time Knowledge
  Graphs for AI Agents*
- **Maintainer:** Zep Software, Inc. (Paul Paliychuk, Preston Rasmussen,
  Daniel Chalef)
- **Why this is here:** The only entry in the study that puts a real,
  typed, **bi-temporal graph** between the agent and its memories — four
  interchangeable Cypher-flavoured backends (Neo4j, FalkorDB, Kuzu,
  Neptune), a single-LLM-call episode-to-graph extraction pipeline with
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
  reading in its own right.

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
- **Maintainer:** Safi Shamsi (`safishamsi`); MIT licensed; PyPI package
  `graphifyy`
- **Why this is here:** The codebase-as-memory bet. Where every other
  entry models *conversational* memory (user / session / agent turns),
  Graphify models the **static corpus an agent works against** as a
  persistent, queryable knowledge graph and exposes it to the agent via
  **MCP** (`graphify/serve.py:1`). The pipeline is single-process and
  legible end-to-end: `detect → extract → build → cluster → analyze →
  report → export` (`ARCHITECTURE.md`), with tree-sitter parsers for
  ~30 languages emitting `{nodes, edges}` dicts that get folded into a
  NetworkX graph, **Leiden community detection** via graspologic with a
  NetworkX-Louvain fallback (`graphify/cluster.py:48-76`), and
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
  dominated by server-backed designs.

### [understand-anything](./understand-anything)
- **Repo:** https://github.com/Egonex-AI/Understand-Anything — *Turn any
  codebase, knowledge base, or docs into an interactive knowledge graph you
  can explore, search, and ask questions about*
- **Maintainer:** Egonex-AI (`Egonex-AI` org; originally Lum1104 / Yuxiang
  Lin, © Infinite Universe, Inc.); MIT; TypeScript plugin v2.8.1
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
  (`packages/core/src/schema.ts:18-78`). (2) **Recall:** Graphify ships an MCP
  query server; Understand-Anything ships *no query API at all* — its eight
  skills (`/understand`, `/understand-chat`, `/understand-diff`, …) instruct the
  agent to **`grep` the single `.understand-anything/knowledge-graph.json`**,
  and an Astro **dashboard** + dependency-ordered **guided tours** serve humans.
  Its ethos is "**graphs that teach > graphs that impress**." (3) **Scope:** a
  `kind: "codebase" | "knowledge"` switch plus five **knowledge** node types
  (article / entity / topic / claim / source) and knowledge edges
  (`cites` / `contradicts` / `builds_on`) point the same machinery at *prose and
  argument* (Karpathy-style LLM wikis, papers) — the only entry in the study
  that models a knowledge base as a first-class graph subject. The schema is
  rich (21 node types, 35 edge types across 8 categories; `types.ts:2-19`) and
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
  the agent loop.

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
  on demand. "Directory recursive retrieval" combines vector search
  with hierarchical navigation. Unifies what every other study entry
  treats as separate concerns: **memories** (user / agent task),
  **resources** (docs, repos), and **skills** (agent instructions)
  are all just nodes in the same tree. ByteDance-scale backing is the
  operational story (Rust core in `crates/`, multi-language CLI, Docker
  deploy). The interesting compare-and-contrast is against Letta's
  git-backed memory mode (which also renders memory as files but on a
  flat-ish layout) and against Graphify (which also collapses
  resource-as-context but as a knowledge graph rather than a tree).

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
  chain on a single fact type; RetainDB enumerates categories).

### [supermemory](./supermemory)
- **Repo:** https://github.com/supermemoryai/supermemory — *The memory
  and context layer for AI*
- **Maintainer:** supermemoryai research lab; MIT
- **Why this is here:** The current benchmark-leaderboard occupant —
  ranked **#1 on LongMemEval, LoCoMo, and ConvoMem** as of mid-2026,
  the three headline harnesses every other entry in this study reports
  against (compare to MemPalace's 96.6% R@5 on LongMemEval and Mem0's
  91.6 / 93.4 on LoCoMo / LongMemEval — the leaderboard moves fast).
  At ~22.7k stars and 1,600+ commits, the most active OSS entry after
  Mem0 / Letta. Design surface is the **universal RAG + memory** bet:
  fact extraction from conversations + user-profile maintenance +
  hybrid search + multimodal ingest + external connectors (Google
  Drive, Notion, GitHub) that sync content into the same store. Scopes
  are `containerTag` (typically a user id) plus optional project tags.
  Claims to handle "temporal changes, contradictions, and automatic
  forgetting" — which makes the comparison against StateBench (the
  yardstick) the obvious next step.

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
  with Cursor, Claude Code, and 20+ LLM providers as the consumer
  surface. The reason it earns a pin despite the licensing caveat:
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
  Beads vs every vector-backed entry. Deep write-up:
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
  attempt. Early-stage (152★, 34 releases, 161 commits) with active
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
  neo as the units under test.

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
