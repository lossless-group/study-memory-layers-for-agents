# Profiles

* [Beads — Profile](Profile__Beads.md) - Not one embedding anywhere in the recall path — retrieval is a SQL view and a graph walk over the agent's own work graph.
* [ByteRover CLI — Profile](Profile__ByteRover.md) - The memory store is a directory of plain-text topic files (HTML since mid-2026, Markdown before) you can cat, grep, and diff, with its own git history. Recall is BM25, not embeddings, and the calling agent does the LLM work.
* [Delta-Mem — Profile](Profile__Delta-Mem.md) - Memory as a dense rank-8 state matrix inside each attention head, updated token by token — the model never looks anything up.
* [Graphify — Profile](Profile__Graphify.md) - The whole memory layer is one in-process NetworkX graph and a `graph.json` you can copy around — no database anywhere.
* [Graphiti — Profile](Profile__Graphiti.md) - `expired_at` is when the engine retired a fact; `invalid_at` is when it stopped being true in the world. Routinely different dates.
* [Hindsight — Profile](Profile__Hindsight.md) - `fact_type` is a database CHECK constraint: world facts and first-person experiences travel through different extraction prompts.
* [Honcho — Profile](Profile__Honcho.md) - Collections are unique on (observer, observed, workspace) — Honcho indexes memory by relationship rather than by user.
* [Letta — Profile](Profile__Letta.md) - The one entry where the agent itself — not an extraction pipeline, not a background summarizer — is the memory manager (V1 server code now on the archive branch).
* [Mem0 — Profile](Profile__Mem0.md) - Despite the graph-memory marketing, the graph is an array of linked memory IDs hanging off spaCy-extracted entities.
* [Meko — Profile](Profile__Meko.md) - A database company's memory layer that runs locally on SQLite, not YugabyteDB: mem0 with its LLM removed, scoped by governed datapacks, with the user's own Claude Code doing extraction.
* [MemPalace — Profile](Profile__MemPalace.md) - The counter-bet: extract nothing. Raw 800-character chunks score 96.6% R@5 on LongMemEval at zero LLM calls.
* [Neo — Profile](Profile__Neo.md) - Familiar high-confidence queries get dialled down to low reasoning effort — memory shows up as a smaller bill, not better accuracy. (Retired 2026-11-08.)
* [OpenViking — Profile](Profile__OpenViking.md) - Memories, resources, and skills collapse into one `viking://` tree. Retrieval used to walk that tree recursively; since v0.4.23 it is one global vector search scoped by it.
* [Paxm — Profile](Profile__Paxm.md) - Paxm is not a memory store — it is routing and durability in front of seven-plus pluggable backends, and it does zero semantic dedup.
* [RetainDB — Profile](Profile__RetainDB.md) - Supersession as an actual indexed column — `validUntil`, `supersededBy` — rather than something the ranker is hoped to handle.
* [StateBench — Profile](Profile__StateBench.md) - A benchmark that audited itself: StateBench 2.0 found its own scorer flagged every correct "not Friday, it moved" answer as a resurrection, and retracted the numbers built on it.
* [Supermemory — Profile](Profile__Supermemory.md) - The most polished client surface in the study, wrapped around an engine that is not in the repo — every SDK points at a hosted API or a closed self-hosted binary.
* [Understand-Anything — Profile](Profile__Understand-Anything.md) - When an LLM authors your graph, the schema layer stops being a validator and becomes a repair shop — alias maps fold synonyms back.
* [Volt — Profile](Profile__Volt.md) - Numeric thresholds decide when to compact, and the model is only ever asked to write summaries. Long-horizon coding as a DB problem.
