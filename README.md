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
| Hybrid vector + entity-link + tight CRUD API | [Mem0](./mem0) |
| Verbatim chunks + hybrid recall, no extraction | [MemPalace](./mempalace) |
| Typed/scoped facts + deterministic supersession + outcome learning | [Neo](./neo) |
| Immutable Postgres log + summary DAG | [Volt](./volt) |
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
