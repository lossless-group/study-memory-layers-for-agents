---
name: Graphify Profile
slug: graphify
upstream: https://github.com/safishamsi/graphify
package: graphifyy (PyPI)
license: Apache-2.0 (relicensed from MIT on 2026-07-22)
maintainer: Safi Shamsi (safishamsi)
study: studies/memory-layers-for-agents
profile_path: studies/memory-layers-for-agents/graphify
profile_kind: cli + library + mcp-server + ai-coding-skill
date_created: 2026-05-21
site_uuid: 85c10be5-eaf3-4454-9c38-91099c35104c
hex_code: nhepv2
date_authored_initial_draft: 2026-05-21
date_authored_current_draft: 2026-11-23
lede: >-
  The whole memory layer is one in-process NetworkX graph and a `graph.json`
  you can copy around — no database anywhere.
summary: >-
  Profile of Graphify (safishamsi) as pinned in the memory-layers-for-agents
  study. It is one half of the study's codebase-as-memory pair with
  Understand-Anything, and the deterministic half: tree-sitter extraction across
  ~40 languages, no LLM in the code write path. Covers the seven-stage pipeline
  (detect/extract/build/cluster/analyze/report/export), the strict nodes-and-
  edges extraction schema, Leiden clustering with a Louvain fallback and its
  hand-tuned community heuristics, the god-nodes and surprising-connections
  analysis, the MCP tool surface with its hand-tuned match-bonus constants,
  watch-mode staleness signalling, and the security chokepoint in security.py.
  Read it alongside Profile__Graphiti (same primitive, opposite premise about
  what an agent needs to remember) and Profile__Understand-Anything
  (deterministic vs LLM-authored graph construction).
publish: true
title: Graphify — Profile
date_modified: 2026-11-23
authors:
  - Michael Staton
status: Reference
type: Profiles
---

# Graphify — Profile

A profile of Graphify as it lives in this study (`studies/memory-layers-for-agents/graphify/`). Cites pinned paths so you can jump to source rather than trust paraphrase. Read alongside [`Profile__Graphiti.md`](./Profile__Graphiti.md) and [`Profile__MemPalace.md`](./Profile__MemPalace.md) — Graphify makes a categorically different bet from both: the "memory" it gives the agent is not conversational history but the *static corpus the agent works against*, modeled as a single in-process knowledge graph.

## TL;DR

Graphify is an **AI coding assistant skill** that turns a folder — code, SQL schemas, R scripts, shell scripts, docs, papers, images, videos — into a queryable knowledge graph, then exposes that graph to the agent over **MCP** (`graphify/serve.py:1`). The agent calls `query_graph`, `get_neighbors`, `get_community`, `god_nodes`, `shortest_path` instead of grepping through files.

The pipeline is single-process and legible end-to-end (`ARCHITECTURE.md`):

```
detect()  →  extract()  →  build_graph()  →  cluster()  →  analyze()  →  report()  →  export()
```

Extraction is **tree-sitter for ~40 languages** (README at the current pin): fifteen `LanguageConfig` entries in `graphify/extract.py:1542-1925`, plus per-language modules under `graphify/extractors/` (Go, Rust, C#, SQL, Solidity, Terraform, Zig, and others) driven by `graphify/extractors/engine.py`. Each extractor returns a strict `{nodes, edges}` dict validated by `graphify/validate.py` before being merged into a NetworkX graph. Clustering is **Leiden via graspologic** (now called through the `graspologic_native` binding directly) with a NetworkX-Louvain fallback (`graphify/cluster.py:122-282`). Every edge carries one of three **confidence labels** — `EXTRACTED` / `INFERRED` / `AMBIGUOUS` (`ARCHITECTURE.md`) — so the agent can reason about how much to trust each relation. Output is three files in `graphify-out/`: `graph.json` (the whole graph), `graph.html` (interactive viewer), `GRAPH_REPORT.md` (the highlights). Optional exports include Obsidian vault, Mermaid call-flow HTML, a Cypher dump (`graphify/export.py:639`), and direct pushes to Neo4j or FalkorDB (`graphify/exporters/graphdb.py`).

**One sentence:** *Graphify treats the project itself as the memory layer — tree-sitter-parsed nodes and edges, Leiden communities, confidence-labelled relations, single `graph.json` artifact, served to any MCP-speaking AI coding assistant via stdio (or Streamable HTTP at the current pin).*

## Why this exists — the design bet

Three observations drive the design:

1. **The corpus is the agent's primary memory.** For coding agents, the question "what does this codebase look like" is asked far more often than "what did the user say last week." Existing memory layers (Mem0, Letta, MemPalace, Graphiti) optimize the second; Graphify optimizes the first.
2. **A graph DB is overkill for project-scale recall.** Graphiti expects Neo4j/FalkorDB/Kuzu/Neptune; MemPalace runs ChromaDB + SQLite. Graphify ships a single `graph.json` plus a NetworkX in-process graph and refuses to add a database. Project-scale graphs (tens of thousands of nodes) fit comfortably in RAM.
3. **MCP is the right interface, not a custom SDK.** Letta exposes an OpenAI-compatible HTTP endpoint; Graphiti exposes a Python library + optional MCP server. Graphify is **MCP-first** — the skill registration step (`graphify install`) wires the stdio server into Claude Code, Codex, OpenCode, Cursor, Gemini CLI, Copilot, Aider, and many more.

## Storage topology — one in-process graph, one JSON file

There is no database. The whole memory layer is:

- **In-process state:** a `networkx.Graph` (or `DiGraph`) built by `graphify/build.py:2443`.
- **On-disk artifact:** `graphify-out/graph.json` — node-link JSON written by `graphify/export.py:342`. Re-opened by the MCP server via `networkx.readwrite.json_graph.node_link_graph` (`graphify/serve.py:83-87`).
- **Cache:** `graphify/cache.py` keeps a per-file stat index (size + mtime_ns + hash) so unchanged files skip re-extraction on subsequent runs — the same trade-off as `make(1)`.
- **Output dir override:** `GRAPHIFY_OUT` env var, now defined once in `graphify/paths.py` and re-exported by `graphify/cache.py:15-19` — supports worktrees and shared-output setups.

Python 3.11.5 through 3.14 required (`pyproject.toml:13`; it was 3.10+ at the May 2026 pin). Licensed **Apache-2.0** since 2026-07-22 (commit `ba7f9ea9`); it was MIT at the previous pin. `LICENSE` is now the Apache text, `LICENSE-MIT` and a `NOTICE` file are kept, and `pyproject.toml` declares `license = "Apache-2.0"`.

This is the **opposite operational posture** from Graphiti (server-required, four pluggable graph DB backends) and even from MemPalace (ChromaDB + SQLite). Graphify denies the database premise and ships the graph as a file you can copy around.

## Extraction — tree-sitter per language, strict schema, three-level confidence

Every language extractor (`graphify/extract.py`) returns the same shape:

```json
{
  "nodes": [
    {"id": "unique_string", "label": "human name", "source_file": "path", "source_location": "L42"}
  ],
  "edges": [
    {"source": "id_a", "target": "id_b", "relation": "calls|imports|uses|...", "confidence": "EXTRACTED|INFERRED|AMBIGUOUS"}
  ]
}
```

(`ARCHITECTURE.md`)

The **confidence label is load-bearing**:

- `EXTRACTED` — the relation is explicit in source (an import statement, a direct call).
- `INFERRED` — a reasonable deduction (call-graph second pass, co-occurrence in context).
- `AMBIGUOUS` — uncertain; surfaced in `GRAPH_REPORT.md` for human review.

This is the same idea Graphiti encodes in `episodes: list[str]` edge provenance, but cheaper — no LLM call needed to label confidence; it falls out of the parser pass.

Languages registered as `LanguageConfig` objects in `graphify/extract.py` include Python, JavaScript, TypeScript, TSX, Java, Groovy, C, C++, Ruby, C#, Kotlin, Scala, PHP, Lua, Swift, plus SQL, Julia, Fortran, and others added through the same dispatch pattern (`graphify/extract.py:1542-1925`), and the rest live as modules under `graphify/extractors/`.

Adding a language is a five-step recipe documented in `ARCHITECTURE.md` (tree-sitter parse → walk nodes → register suffix → add fixture → write test). The pattern is shallow enough that a new language extractor is one PR, not a feature. Since 0.9.x most new languages land as their own module under `graphify/extractors/` rather than in `extract.py` (`graphify/extractors/MIGRATION.md`).

## Build & dedup — fold extractions into a single NetworkX graph

`graphify/build.py:2443` (`build`) merges per-file `{nodes, edges}` dicts into a graph, normalising IDs (`normalize_id`, now in `graphify/ids.py` and imported at `build.py:33`) and source paths (`_norm_source_file`, line 870). `deduplicate_by_label` (line 2561) collapses obvious aliases before community detection.

`graphify/build.py:2902` (`build_merge`) handles incremental merges so a cache-hit re-run only needs to fold in the changed files' extractions. `graphify/build.py:3452` (`prefix_graph_for_global`) namespaces a single repo's nodes with a `repo_tag` prefix so multiple repos can be merged into a single global graph — the seed of the "everything I work on in one graph" use case.

## Clustering — Leiden, with a Louvain fallback

```python
from graspologic.partition import leiden
# ...
result = leiden(stable, **kwargs)
# Fallback: networkx louvain (available since networkx 2.7).
communities = nx.community.louvain_communities(stable, **kwargs)
```

(`graphify/cluster.py:122-282`; at the current pin `_native_leiden` calls `graspologic_native.leiden()` first, then `graspologic.partition.leiden`, then Louvain.)

`cluster()` returns `{community_id: [node_ids]}` and does not mutate the graph (`ARCHITECTURE.md`). Community IDs are stable across runs (`0 = largest community after splitting`, per `graphify/cluster.py:349`), and since 0.9.x `graph.json` output is deterministic run to run because the CLI pins `PYTHONHASHSEED` (CHANGELOG). Communities feed two downstream features: the `get_community` MCP tool, and PR-impact analysis ("which communities does this PR touch").

`_MIN_SPLIT_SIZE`, `_COHESION_SPLIT_MIN_SIZE` (`graphify/cluster.py:287-289`), a majority-vote rule for reattaching excluded hubs (`cluster.py:355-362`), and, new since the last pin, an `ambiguous_scale` of 0.5 that halves the weight of `AMBIGUOUS` edges before partitioning (`cluster.py:344, 363-367`) keep the partition readable instead of producing a long tail of tiny communities — the kind of fiddly heuristic you only write after staring at real outputs.

## Analyze — god nodes, surprising connections, suggested questions

`graphify/analyze.py` produces the structured analysis the report renders:

- `god_nodes(G, top_n=10)` (`graphify/analyze.py:119`) — most-connected nodes, the project's "central abstractions."
- `surprising_connections(...)` (line 158) — cross-file, cross-language, or cross-community edges scored by an explicit `_surprise_score` rubric (line 253). The intuition: same-folder edges are boring; long-distance edges might be where the architecture is leaking. This is closer in spirit to "what should I read next" than "what is the answer."
- `_cross_language(src_a, src_b)` (line 53) — explicit signal for edges that bridge language boundaries, surfaced as a distinct surprise category.

The report (`graphify/report.py`) renders these into `GRAPH_REPORT.md` alongside suggested questions an agent might ask.

## Recall surface — MCP tools, not a Python SDK

`graphify/serve.py:3584` (`serve`) boots a stdio MCP server; `serve_http` (`serve.py:3744`) serves the same tools over Streamable HTTP so one process can host a graph for a team. `start_server` from the earlier pin no longer exists. The tools exposed (`graphify/serve.py:2986-3140`):

| Tool | Purpose |
|---|---|
| `query_graph` | BFS or DFS search around a natural-language question, capped by token budget. `mode`, `depth`, `token_budget`, `context_filter` are all knobs. |
| `get_node` | Full details for a node by label or ID. |
| `get_section` | New since the May pin: return the text of a Markdown section (a heading node and its subsections) or a whole page, read from source. |
| `get_neighbors` | Direct neighbours of a node, optionally filtered by relation. |
| `get_community` | All nodes in a community by community ID. |
| `god_nodes` | Top-N most connected nodes. |
| `graph_stats` | Node/edge/community/confidence summary. |
| `shortest_path` | Shortest path between two concepts, with `max_hops` cap. |
| `list_prs` / `get_pr_impact` / `triage_prs` | GitHub-aware: list open PRs with CI + review state + which graph communities they touch, assess blast radius before starting work. |

The server also exposes MCP resources (`graphify://report`, `stats`, `god-nodes`, `surprises`, `audit`, `questions`; `serve.py:3456-3461`).

Two ranking knobs in the query path are worth flagging (`graphify/serve.py:391-394`):

```python
_EXACT_MATCH_BONUS = 1000.0
_PREFIX_MATCH_BONUS = 100.0
_SUBSTRING_MATCH_BONUS = 1.0
_SOURCE_MATCH_BONUS = 0.5
```

Hand-tuned constants — the same kind of "we stared at real outputs" residue visible in the clustering code. IDF weights for query terms are cached on the graph object (`_compute_idf`, `graphify/serve.py:515-523`).

The MCP transport hardening is worth a look on its own (`graphify/serve.py:2836-2842`): blank-line filtering on stdin to survive Claude Desktop and other clients that emit framing whitespace.

## Watch mode — staleness signalling, not push notifications

At the current pin, `graphify/watch.py` splits by file type (`watch()`, `watch.py:3745-3754`). Code-only changes re-run AST extraction and rebuild immediately, with no LLM. Doc, paper, and image changes write a `graphify-out/needs_update` flag and tell the user to run `/graphify --update`, because those need LLM extraction; `check_update` reports the flag (`watch.py:3589-3600`). The MCP server no longer polls a flag. It hot-reloads `graph.json` when the file's `(st_mtime_ns, st_size)` key changes (`graphify/serve.py:180-201`), serving the old graph while a reload is in flight. No file-system events propagate through the MCP transport. Cheap, correct, easy to reason about.

## Security model

`graphify/security.py` is the chokepoint for external input (`ARCHITECTURE.md`):

- URLs → `validate_url()` (http/https only) + `_NoFileRedirectHandler` (blocks `file://` redirects), plus SSRF-guarded HTTP(S) connection classes that block private IPs at connect time (`graphify/security.py:83-258`).
- Fetched content → `safe_fetch()` / `safe_fetch_text()` (size cap + timeout).
- Graph file paths → `validate_graph_path()` (must resolve inside `graphify-out/`).
- Node labels → `sanitize_label()` (strips control chars, caps 256 chars, HTML-escapes).

The HTML-escape on labels is specifically motivated as protection against an attacker who controls a source file injecting markup that lands in MCP tool output (`graphify/serve.py:1435-1439`, F-010). This is a class of issue that conversational-memory entries in the study don't have to think about — but a project-as-memory entry, where the corpus is by definition adversarial-tolerant, does.

## On-disk shape — `graph.json` as the canonical artifact

```
graphify-out/
├── graph.html        interactive viewer (filter / search)
├── GRAPH_REPORT.md   highlights: key concepts, surprises, suggested questions
└── graph.json        full graph — query anytime without re-reading files
```

The format is NetworkX's node-link JSON. `graph.json` is the source of truth; the HTML and the report are derived. The `Cypher` export (`graphify/export.py:639` — `to_cypher`) is a one-way dump for users who want to load the graph into Neo4j — explicitly *not* a dependency.

The "could a non-AI program parse it cleanly" check from the study's working questions is satisfied trivially here: `graph.json` is plain JSON, schema documented in `ARCHITECTURE.md`.

## How it compares to the rest of the study

| Dimension | Graphify | Graphiti | MemPalace | Mem0 | Letta |
|---|---|---|---|---|---|
| What is the memory of? | Static corpus (codebase / docs) | Conversational episodes | Conversational chunks | User / session facts | Agent core context + archival |
| Storage primitive | NetworkX in-process + `graph.json` | Cypher graph DB (4 backends) | ChromaDB + SQLite | Vector + KV + optional graph | Postgres + pgvector |
| Server required? | No (stdio MCP) | Yes | No (embedded) | Either | Yes (FastAPI + Postgres) |
| Write trigger | `/graphify .` on the folder | Per Episode (LLM-extracted) | Per chunk (verbatim, no LLM) | Per turn (LLM-summarised) | Per turn (agent self-edits) |
| Recall API | MCP tools (`query_graph`, etc.) | Python lib + REST + MCP | Python lib | REST + Python lib | OpenAI-compatible + REST + WS |
| Confidence on edges | Per-edge label (EXTRACTED / INFERRED / AMBIGUOUS) | Per-edge episode provenance | n/a (verbatim) | n/a | n/a |
| Temporal model | None — it's a snapshot | Bi-temporal (`valid_at`/`invalid_at` × `created_at`/`expired_at`) | Per-chunk timestamps | Per-record updated_at | Per-message timestamps |
| Code-aware? | Yes (tree-sitter, ~40 langs) | No | No | No | No |

The cleanest pairing is **Graphify vs Graphiti**:

- Same primitive in spirit (typed knowledge graph), different premise about what the agent needs to remember.
- Graphiti expects you to *teach* the graph; Graphify expects to *derive* it from a corpus you already have.
- Graphiti's bi-temporal model is the headline; Graphify's confidence labels are the headline. Both are forms of provenance, but at different granularities.

The cleanest pairing on **operational posture** is **Graphify vs MemPalace**: both refuse the server-required default; both ship as a Python package; both treat the on-disk artifact (Chroma collections + SQLite, or `graph.json`) as the canonical state. The architecture stance is "the memory layer should be a file."

## Working-checklist answers

Per the study's reading rubric:

- **Storage topology.** In-process NetworkX graph + on-disk `graph.json`. No external DB.
- **Write policy.** Batch — re-run `/graphify` on the folder, cache skips unchanged files. Not append-only at the relation level; the graph is rebuilt and overwritten (with a backup-protection check, `backup_if_protected`, `graphify/export.py:42`).
- **Scopes & namespacing.** Per-folder graphs; multi-repo via `prefix_graph_for_global` (`graphify/build.py:3452`). No user/session/agent scoping — the unit is the project.
- **Schema of a memory.** Strict `{nodes, edges}` dict, with `id`/`label`/`source_file`/`source_location` on nodes and `source`/`target`/`relation`/`confidence` on edges (`ARCHITECTURE.md`). Validated by `graphify/validate.py`.
- **Recall surface.** MCP tools — BFS/DFS query, neighbor lookup, community fetch, god nodes, shortest path. IDF + exact/prefix/substring/source-match scoring (`graphify/serve.py:391-394`).
- **Eviction & compaction.** None — rebuild model. Code changes rebuild automatically under `graphify watch`; doc changes set a `needs_update` flag. The new `graphify reflect` lessons file (below) applies time-decay to session outcomes, which is the closest thing to eviction.
- **Serialization on disk.** Plain NetworkX node-link JSON. Schema in `ARCHITECTURE.md`. Cypher export available but optional.
- **Operational story.** Local-first. Stdio MCP server by default, optional Streamable HTTP listener (`serve_http`, binds 127.0.0.1:8080 by default with an optional API key), no database. One process, one folder, one `graph.json`.

## Changes since the previous pin (2026-05-20 → 2026-11-23)

Old pin `076e6b7c06d0` (0.8.14), new pin `a3db62f680cd` (0.9.115, CHANGELOG dated 2026-11-21), 2,666 commits on the `v8` branch, about 1,200 files changed. Roughly 100 PyPI releases landed in range, starting with 0.9.0 on 2026-06-28. The README now names `Graphify-Labs/graphify` as the official repository; the `safishamsi/graphify` remote this study pins still resolves. Line citations above have been re-located to the new pin. The thesis holds: one NetworkX graph and one `graph.json`, no database required.

- **Relicensed MIT → Apache-2.0** on 2026-07-22 (`ba7f9ea9`). `LICENSE` is the Apache text; `LICENSE-MIT` and `NOTICE` are kept; `pyproject.toml` declares `license = "Apache-2.0"` and lists all three as license files. The CHANGELOG entry says contributions made before the change remain available under MIT. The frontmatter `license:` key of this profile still says MIT.
- **Breaking node-ID change in 0.9.0.** IDs now use the full repo-relative path, so same-named files in different folders no longer collapse into one node. Existing graphs migrate on the next build; nodes that had already collided come back only after `graphify extract --force` (CHANGELOG, 0.9.0). Anyone persisting node IDs outside `graph.json` (Neo4j, GraphML layouts) has to re-import.
- **A work-memory loop on top of the graph.** `graphify save-result` files Q&A outcomes (`useful`, `dead_end`, `corrected`; `graphify/ingest.py:319`) into `graphify-out/memory/`, and `graphify reflect` turns them into `graphify-out/reflections/LESSONS.md`: preferred, tentative, and contested source nodes, known dead ends, and corrections, scored with a time-decayed signal and no LLM (`graphify/reflect.py:1-25`). This is the first feature in Graphify that remembers what the agent learned, not just what the corpus contains, and it moves Graphify closer to the conversational-memory entries in this study.
- **MCP over HTTP and new surfaces.** `serve_http` (`graphify/serve.py:3744`), a `get_section` tool, six MCP resources, and direct Neo4j/FalkorDB push (`graphify/exporters/graphdb.py`).
- **Extraction moved into `graphify/extractors/`** with a shared engine (`extractors/engine.py`, `extractors/resolution.py`) and ~40 languages. Much of the CHANGELOG is cross-file call-resolution work per language that removes false high-confidence `calls` edges.
- **Confidence got finer.** INFERRED edges carry a relation-keyed `confidence_score`; `shortest_path` reports the weakest hop as the path's confidence; when duplicate edges collapse, the weaker copy's provenance is kept in `also_seen`; AMBIGUOUS edges are down-weighted in clustering (CHANGELOG 0.9.x; `graphify/cluster.py:363-367`).
- **Watch and serve.** Code changes rebuild automatically; the server hot-reloads `graph.json` by mtime and size (`graphify/serve.py:180-201`).
- **Python floor raised** to 3.11.5 (`pyproject.toml:13`), and Leiden now calls `graspologic_native` directly to avoid graspologic's heavy imports.

## Pointers worth following

- `ARCHITECTURE.md` — the seven-stage pipeline, the extraction schema, the security model.
- `graphify/extract.py:1542-1925` and `graphify/extractors/` — the `LanguageConfig` registry and the per-language modules. Reading two or three of these in sequence is the fastest way to understand what a tree-sitter-based extractor actually does.
- `graphify/cluster.py:122-545` — Leiden with Louvain fallback, plus the empirical heuristics for community splitting and hub reattachment.
- `graphify/analyze.py:158-490` — the surprise-scoring rubric. Less obvious than god nodes; more interesting than god nodes.
- `graphify/serve.py:2986-3140` — MCP tool declarations.
- `graphify/reflect.py` — the deterministic lessons file built from saved Q&A outcomes. Read these to see what the agent actually *asks* of the graph.
- `graphify/security.py` + `SECURITY.md` — the threat model. The fact that this exists at all is part of what distinguishes a project-as-memory entry from a conversation-as-memory entry.

## Open questions for the study

- Does the static-snapshot model break down at a certain repo size? `graph.json` for a 1M-LOC monorepo would not be small.
- How well does `prefix_graph_for_global` actually compose across repos in practice? A cross-repo "find me anyone calling this function" use case is where this stops being a single-folder tool and starts being a portfolio-of-projects memory layer — Lossless-shaped.
- How does the confidence-label discipline survive in the presence of LLM-driven extractors? Code is tree-sitter, but docs, papers, and images go through LLM extraction, and the CHANGELOG since the May pin shows the cost: many fixes fold "drifted" model output (`"high"`, `"confirmed"`, lower-case `extracted`) back onto the enum, and an LLM-claimed `EXTRACTED` code node is now evidence-checked before it is accepted. INFERRED edges also carry a numeric `confidence_score` keyed to the relation.
- The `triage_prs` / `get_pr_impact` tools are an interesting boundary — knowledge graph as input to *decision-making about future work*, not just recall. Worth watching whether other entries grow toward this surface.
