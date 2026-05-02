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
