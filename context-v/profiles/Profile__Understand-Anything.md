---
name: Understand-Anything Profile
slug: understand-anything
upstream: https://github.com/Egonex-AI/Understand-Anything
plugin_version: 2.8.1 (.claude-plugin/plugin.json)
license: MIT
maintainer: Egonex-AI (org); originally Lum1104 / Yuxiang Lin; © Infinite Universe, Inc.
study: studies/memory-layers-for-agents
profile_path: studies/memory-layers-for-agents/understand-anything
profile_kind: ai-coding-plugin (multi-agent) + knowledge-graph + astro-dashboard
pinned_sha: 7f5a717694d3a94f19f523b375c777eb21548ff5
date_created: 2026-06-19
site_uuid: a511f50d-6b08-413b-ba1f-466cd90511f8
hex_code: 5npjhb
date_authored_initial_draft: 2026-06-19
date_authored_current_draft: 2026-06-19
lede: >-
  When an LLM authors your graph, the schema layer stops being a validator and
  becomes a repair shop — alias maps fold synonyms back.
summary: >-
  Profile of Understand-Anything (Egonex-AI) as pinned in the
  memory-layers-for-agents study. It is the probabilistic half of the study's
  codebase-as-memory pair with Graphify — same thesis, opposite execution on
  extraction, recall, and scope. Covers the nine-agent LLM pipeline over a
  tree-sitter skeleton spanning 42 language configs, the 21-node-type and
  35-edge-type vocabulary including the knowledge family that models prose and
  argument, the Zod schema plus alias-normalization layer, the four plain-JSON
  files under .understand-anything with no database or server, fingerprint-based
  incremental re-analysis with a post-commit hook, path sanitization to avoid
  leaking home directories through the dashboard, and the eight skills that make
  up the recall surface. The alias-map section is the most transferable idea in
  the file; the kind-knowledge path is the one worth probing against the
  Lossless corpus.
publish: true
---

# Understand-Anything — Profile

A profile of Understand-Anything as it lives in this study
(`studies/memory-layers-for-agents/understand-anything/`). Cites pinned paths so
you can jump to source rather than trust paraphrase. Read **alongside**
[`Profile__Graphify.md`](./Profile__Graphify.md) — these two are the study's
clearest matched pair: both make the *codebase-as-memory* bet (model the static
corpus an agent works against as a knowledge graph, not conversational history),
both are tree-sitter-backed, both are MIT, both install as multi-platform AI
coding plugins. They diverge on almost everything below the thesis, which is
what makes the pairing worth reading.

## TL;DR

Understand-Anything turns a folder — code, docs, knowledge bases, papers — into a
**single JSON knowledge graph** (`.understand-anything/knowledge-graph.json`)
through a **multi-agent LLM pipeline**, then exposes that graph three ways: an
Astro **dashboard** for humans, eight **skills** the agent invokes (`/understand`,
`/understand-chat`, `/understand-diff`, …), and **guided tours** for onboarding.

Verbatim tagline (`README.md:4`):

> *Turn any codebase, knowledge base, or docs into an interactive knowledge graph
> you can explore, search, and ask questions about.*

And the design ethos, also from the README: **"Graphs that teach > graphs that
impress."** That sentence is the whole positioning — the artifact is optimized for
human onboarding and agent comprehension, not for being a queryable database.

The pipeline is **agentic**, not deterministic. Nine sub-agents
(`understand-anything-plugin/agents/`) each own one stage:

```
project-scanner → file-analyzer → architecture-analyzer → domain-analyzer
                → article-analyzer → tour-builder → graph-reviewer
                → assemble-reviewer  (knowledge-graph-guide is the read-side helper)
```

tree-sitter does the deterministic parse (42 language configs in
`packages/core/src/languages/configs/`), and the LLM agents layer summaries,
intent, architectural layers, and business domains on top. The crucial twist:
because an LLM emits the graph, the output is **validated and canonicalized
against a Zod schema** (`packages/core/src/schema.ts`) with explicit alias maps
that fold the synonyms LLMs reach for (`fn`/`method` → `function`,
`extends` → `inherits`) back to canonical types.

**One sentence:** *Understand-Anything is a multi-agent LLM pipeline that extracts
a 21-node-type / 35-edge-type knowledge graph from any corpus, normalizes the
LLM's output against a Zod schema, persists it as one `knowledge-graph.json`, and
serves it to humans (Astro dashboard, guided tours) and agents (skills that Grep
the JSON) — optimized to teach, not to be queried.*

## Why this exists — the design bet

Three observations drive the design:

1. **The corpus is the agent's primary memory** — same opening premise as
   Graphify. The hard question for a coding (or research) agent is "what does this
   project *mean*," asked far more than "what did the user say last week."
2. **Deterministic parsing + LLM semantics, not one or the other.** tree-sitter
   gives a faithful structural skeleton; LLM agents add the layer the parser can't
   — English summaries, architectural layers, business domains, "why." The graph
   is a *hybrid* artifact, and the README says so explicitly.
3. **A graph is for understanding, not for querying.** Where Graphify ships an MCP
   server with `query_graph`/`shortest_path`/`god_nodes`, Understand-Anything ships
   a *dashboard* and *guided tours*. Recall, for the agent, is "Grep this JSON
   file" — the skill (`skills/understand-chat/SKILL.md`) literally instructs the
   agent to `grep -i` the graph file before reading. The bet is that comprehension
   beats programmatic traversal for the onboarding/explanation use case.

The fourth, quieter bet — and the one that most distinguishes it from Graphify —
is **`kind: "codebase" | "knowledge"`** (`types.ts:91`, `schema.ts:423`). The same
graph machinery is pointed at *knowledge bases and papers* (Karpathy-style LLM
wikis), with a dedicated `article-analyzer` agent and five **knowledge node types**
(article / entity / topic / claim / source) plus knowledge edges
(`cites` / `contradicts` / `builds_on` / `exemplifies`). This is the only entry in
the study that treats *a corpus of prose/argument* as a first-class graph subject,
not just code.

## Storage topology — one JSON file, no database, no server

The entire memory layer is a directory of plain-JSON files at the project root
(`packages/core/src/persistence/index.ts:7-11`):

```
.understand-anything/
├── knowledge-graph.json   the graph (the source of truth)
├── meta.json              AnalysisMeta — lastAnalyzedAt, gitCommitHash, version, analyzedFiles, theme
├── fingerprints.json      per-file fingerprints for incremental re-analysis
└── config.json            ProjectConfig — auto-update opt-in, language preference
```

There is **no database and no long-running server**. Like Graphify, it denies the
graph-DB premise (contrast Graphiti's Neo4j/FalkorDB/Kuzu/Neptune) and ships the
graph as a file you can copy around. Unlike Graphify (NetworkX in-process graph +
`graph.json`, Python), the canonical state here is *purely* the JSON on disk;
there is no in-process graph object that outlives a skill invocation — the agent
re-reads (greps) the file each time.

The Astro dashboard (`packages/dashboard/`) is a *viewer* booted on demand
(`/understand-dashboard`), not a persistent query backend.

## The graph schema — 21 node types, 35 edge types, Zod-validated

Root shape (`packages/core/src/types.ts:91-100`):

```ts
interface KnowledgeGraph {
  version: string;
  kind?: "codebase" | "knowledge";
  project: ProjectMeta;     // name, languages[], frameworks[], description, analyzedAt, gitCommitHash
  nodes: GraphNode[];
  edges: GraphEdge[];
  layers: Layer[];          // logical groupings (architectural layers)
  tour: TourStep[];         // ordered onboarding walkthrough
}
```

**Node** (`types.ts:39-51`):

```ts
interface GraphNode {
  id: string;               // type-prefixed, e.g. file:path, function:path:name, article:path
  type: NodeType;           // one of 21
  name: string;
  filePath?: string;
  lineRange?: [number, number];
  summary: string;          // LLM-authored
  tags: string[];
  complexity: "simple" | "moderate" | "complex";
  languageNotes?: string;   // "language concept callouts" for the teaching mode
  domainMeta?: DomainMeta;       // entities, businessRules, crossDomainInteractions, entryPoint
  knowledgeMeta?: KnowledgeMeta; // wikilinks, backlinks, category, content
}
```

The **21 node types** (`types.ts:2-7`) span four families:

| Family | Types |
|---|---|
| Code (5) | file, function, class, module, concept |
| Non-code (8) | config, document, service, table, endpoint, pipeline, schema, resource |
| Domain (3) | domain, flow, step |
| Knowledge (5) | article, entity, topic, claim, source |

**Edge** (`types.ts:54-61`): `{ source, target, type, direction, description?, weight }`
where `direction` is `forward | backward | bidirectional` and `weight` is `0–1`.
The **35 edge types** (`types.ts:10-19`) are grouped into 8 categories: Structural
(`imports`, `inherits`, `implements`), Behavioral (`calls`, `publishes`,
`middleware`), Data flow (`reads_from`, `writes_to`, `transforms`), Dependencies
(`depends_on`, `tested_by`), Semantic (`related`, `similar_to`), Infrastructure
(`deploys`, `serves`, `provisions`, `triggers`), Schema/Data (`migrates`,
`routes`, `defines_schema`), Domain (`contains_flow`, `flow_step`, `cross_domain`),
and **Knowledge** (`cites`, `contradicts`, `builds_on`, `exemplifies`,
`categorized_under`, `authored_by`).

The Knowledge node-and-edge family is the schema's tell: this is a graph format
designed to also hold *a corpus of argument* — claims that cite sources,
articles that build on or contradict each other — not just a call graph.

## The novel bit — Zod schema + LLM alias normalization

Because the graph is **LLM-extracted**, the output drifts: an agent writes `fn`
when the schema wants `function`, `extends` when it wants `inherits`. Graphify
never has this problem (its tree-sitter extractors emit canonical types directly).
Understand-Anything solves it with two alias maps in `schema.ts`:

- `NODE_TYPE_ALIASES` (`schema.ts:18-78`) — ~45 synonyms folded to canonical node
  types: `func`/`fn`/`method` → `function`; `interface`/`struct` → `class`;
  `container`/`deployment`/`pod` → `service`; `terraform`/`infra` → `resource`;
  `paper`/`reference`/`raw` → `source`; `decision`/`thesis`/`assertion` → `claim`.
  A pointed comment notes `process` is *deliberately excluded* as an alias
  ("ambiguous with OS/Node.js process").
- `EDGE_TYPE_ALIASES` (`schema.ts:81+`) — `invokes` → `calls`, `uses`/`requires`
  → `depends_on`, `references`/`cites_source` → `cites`, `conflicts_with`/
  `disagrees_with` → `contradicts`, `elaborates`/`refines` → `builds_on`.

This is the load-bearing engineering insight of the whole repo for the study:
**when an LLM authors your memory graph, the schema layer must be a
canonicalizer, not just a validator.** It's the direct counterpart to Graphify's
per-edge confidence labels (`EXTRACTED`/`INFERRED`/`AMBIGUOUS`) — both are
disciplines for trusting a machine-built graph, but they target opposite failure
modes: Graphify labels *how sure* the deterministic parser is; Understand-Anything
*repairs* what the probabilistic extractor got loosely right.

## Extraction — a multi-agent LLM pipeline over a tree-sitter skeleton

The nine sub-agents (`understand-anything-plugin/agents/`):

| Agent | Role |
|---|---|
| `project-scanner.md` | discover files; detect languages & frameworks |
| `file-analyzer.md` | per-file functions/classes/imports → graph nodes & edges |
| `architecture-analyzer.md` | identify architectural **layers** |
| `domain-analyzer.md` | extract business **domains** and process **flows** |
| `article-analyzer.md` | the `kind: knowledge` path — prose → article/claim/source nodes |
| `tour-builder.md` | build the dependency-ordered guided **tour** |
| `graph-reviewer.md` | validate graph integrity |
| `assemble-reviewer.md` | final assembly review |
| `knowledge-graph-guide.md` | read-side helper for agents querying the graph |

tree-sitter supplies the deterministic substrate: **42 language configs**
(`packages/core/src/languages/configs/`) — and notably they span *far past code*:
not just `python.ts`/`rust.ts`/`typescript.ts` but `terraform.ts`,
`kubernetes.ts`, `dockerfile.ts`, `github-actions.ts`, `openapi.ts`,
`graphql.ts`, `protobuf.ts`, `markdown.ts`, `restructuredtext.ts`, `csv.ts`. The
non-code/infrastructure node types in the schema are not aspirational — there's a
parser behind each. A `tree-sitter-dart-wasm` package ships a WASM grammar for the
one language without a native build.

## Recall surface — skills that Grep the JSON, plus a dashboard for humans

There is **no programmatic query API**. This is the single sharpest contrast with
Graphify, which exposes `query_graph`/`get_neighbors`/`get_community`/`god_nodes`/
`shortest_path` as MCP tools. Understand-Anything's recall is **eight skills**
(`understand-anything-plugin/skills/`) the agent invokes as slash-commands:

| Skill | What it does |
|---|---|
| `understand` | run the full pipeline; build the graph |
| `understand-chat` | answer questions about the codebase from the graph |
| `understand-dashboard` | boot the Astro interactive viewer |
| `understand-diff` | diff-impact / ripple analysis of a change |
| `understand-domain` | explore business domains & flows |
| `understand-explain` | explain a specific node/area |
| `understand-knowledge` | the `kind: knowledge` analysis path (wikis, papers) |
| `understand-onboard` | guided onboarding via the tour |

The recall *mechanism* is striking in its plainness. `understand-chat/SKILL.md`
instructs the agent to **`grep -i` the JSON file for query keywords, read only the
sections it needs, and follow `imports`/`calls` edges for dependency chains** —
explicitly "don't dump the entire graph into context." So "recall" is grep over a
local JSON document, with the graph's structure (type-prefixed IDs, edge types) as
the index. Cheap, transparent, no server — and a deliberate bet that a good
schema + grep beats a query engine for the comprehension use case.

The human-facing recall surface is the **Astro dashboard** (`packages/dashboard/`)
— pan/zoom/search over the graph, guided tours, diff-impact view. "Graphs that
teach" is realized here: the primary consumer is a developer onboarding to an
unfamiliar codebase, with the agent as a co-reader.

## Incremental updates — fingerprints + post-commit hook

Re-analysis is incremental, keyed on **fingerprints**
(`packages/core/src/fingerprint.ts`). Each file gets a `FileFingerprint`
(`contentHash` + structured fingerprints of its functions, classes, imports,
exports) persisted to `fingerprints.json`; unchanged files skip re-analysis — the
same `make(1)`-style staleness check Graphify implements with its file-stat cache.

A **post-commit hook** (`hooks/auto-update-prompt.md`, `hooks/hooks.json`) plus
`config.json`'s auto-update opt-in keeps the graph fresh as the repo changes — so
the memory layer tracks the corpus without a manual rebuild, gated behind an
explicit opt-in.

## Security model — path sanitization before persistence

`persistence/index.ts` has a load-bearing `sanitiseFilePaths()` pass
(`index.ts:42-72`) run on every `saveGraph()`. Because the analysis agents emit
**absolute** paths (`/Users/alice/company/src/auth.ts`), and the graph is later
served by the dashboard, writing them verbatim would leak the developer's home
directory, username, and company directory layout. The fix: paths inside the
project root become relative; absolute-but-outside paths are reduced to their
basename. This is the same class of concern Graphify addresses with
`sanitize_label()` / `validate_graph_path()` — a *corpus-as-memory* tool ingests
adversarial-tolerant input and serves it onward, so it has to think about
injection and leakage in a way conversational-memory entries don't.

## On-disk shape — `knowledge-graph.json` as the canonical artifact

`knowledge-graph.json` is plain, pretty-printed JSON
(`writeFileSync(..., JSON.stringify(sanitised, null, 2))`, `index.ts:79`),
validated on load against the Zod schema (`loadGraph(..., { validate: true })`).
The study's "could a non-AI program parse it cleanly" check passes trivially: it's
a documented JSON object (`version`, `kind`, `project`, `nodes[]`, `edges[]`,
`layers[]`, `tour[]`), and the `understand-chat` skill's reliance on plain `grep`
is the proof — no special reader required.

## How it compares to the rest of the study

| Dimension | Understand-Anything | Graphify | Graphiti | MemPalace |
|---|---|---|---|---|
| What is the memory of? | Static corpus — code **or** knowledge base / papers | Static corpus (code / docs) | Conversational episodes | Conversational chunks |
| Extraction | **Multi-agent LLM** over tree-sitter skeleton | Deterministic tree-sitter | LLM episode→graph | Verbatim (no extraction) |
| Storage primitive | One `knowledge-graph.json` (no in-proc graph) | NetworkX in-process + `graph.json` | Cypher graph DB (4 backends) | ChromaDB + SQLite |
| Server required? | No | No (stdio MCP) | Yes | No (embedded) |
| Recall API | **Skills that `grep` the JSON** + Astro dashboard | MCP tools (`query_graph`, …) | Python lib + REST + MCP | Python lib |
| Schema discipline | Zod validate **+ LLM alias normalization** | Strict `{nodes,edges}` + confidence labels | Pydantic-typed entities | n/a (verbatim) |
| Node/edge vocabulary | 21 node types / 35 edge types incl. a **knowledge** family | code-graph nodes + 3-level confidence edges | typed entities + bi-temporal edges | n/a |
| Temporal model | Snapshot + `gitCommitHash` + fingerprints | None — snapshot | Bi-temporal | Per-chunk timestamps |
| Code-aware? | Yes (42 tree-sitter configs incl. infra/docs) | Yes (~30 langs) | No | No |
| Headline framing | "Graphs that **teach**" — onboarding/dashboard | "Project is the memory layer" — MCP recall | Real-time temporal KG | Best-benchmarked verbatim |
| Stack / license | TypeScript / MIT | Python / MIT | Python / Apache-2.0 | Python / MIT |

The cleanest pairing is **Understand-Anything vs Graphify** — same thesis,
opposite executions on three axes:

- **Extraction:** Graphify is *deterministic* (tree-sitter emits the graph
  directly, confidence falls out of the parser). Understand-Anything is
  *probabilistic* (LLM agents author the graph, then a Zod+alias layer repairs it).
  This is the study's clearest worked example of the two ways to build a code KG.
- **Recall:** Graphify is *query-first* (MCP tools, BFS/DFS, god-nodes, IDF-scored
  search). Understand-Anything is *comprehension-first* (grep the JSON, a dashboard,
  guided tours). "Graphs that query" vs "graphs that teach."
- **Scope:** Graphify stays in code/docs/media as one code-shaped graph.
  Understand-Anything's `kind: "knowledge"` + the article/claim/source/cites/
  contradicts vocabulary extends the same machinery to *prose and argument* — the
  only study entry that models a knowledge base as a first-class graph subject.

## Working-checklist answers

Per the study's reading rubric:

- **Storage topology.** Four plain-JSON files under `.understand-anything/`; the
  graph is `knowledge-graph.json`. No DB, no server, no persistent in-process graph.
- **Write policy.** Batch — run `/understand`; a multi-agent LLM pipeline rebuilds
  the graph, normalized against a Zod schema. Incremental re-runs skip unchanged
  files via `fingerprints.json`; an opt-in post-commit hook auto-updates.
- **Scopes & namespacing.** Per-project graph. Type-prefixed node IDs
  (`file:path`, `function:path:name`, `article:path`). `layers[]` give logical
  (architectural) grouping; `domain` nodes give business grouping. No
  user/session/agent scope — the unit is the project/corpus.
- **Schema of a memory.** `GraphNode` = id/type/name/filePath?/lineRange?/summary/
  tags/complexity/languageNotes?/domainMeta?/knowledgeMeta? (`types.ts:39`).
  `GraphEdge` = source/target/type/direction/description?/weight (`types.ts:54`).
  21 node types, 35 edge types, Zod-validated with alias normalization.
- **Recall surface.** Eight skills (`/understand-chat` etc.) that **`grep` the JSON
  file** + follow edges; an Astro dashboard and guided tours for humans. No
  programmatic query API.
- **Eviction & compaction.** None — rebuild model. Fingerprints drive incremental
  re-analysis; the hook keeps it fresh. No TTL, no summarizer-compaction of the
  graph itself.
- **Serialization on disk.** Pretty-printed JSON, schema-validated on load,
  absolute paths sanitized to relative/basename before write (`index.ts:42-79`).
- **Operational story.** Local-first. Installs as a plugin across Claude Code,
  Codex, Cursor, Copilot, Gemini CLI, OpenCode, Vibe CLI, Trae. The agent is one
  consumer; the human (dashboard, tours) is an equal first-class consumer.

## Pointers worth following

- `packages/core/src/schema.ts:18-78` — `NODE_TYPE_ALIASES` / `EDGE_TYPE_ALIASES`.
  The single most study-relevant file: how you canonicalize an LLM-authored graph.
- `packages/core/src/types.ts:2-100` — the full node/edge vocabulary and the
  `KnowledgeGraph` root type. Read the Knowledge family to see the prose-graph bet.
- `packages/core/src/persistence/index.ts:42-79` — `sanitiseFilePaths` + `saveGraph`.
  The leakage threat model and the canonical on-disk shape.
- `understand-anything-plugin/skills/understand-chat/SKILL.md` — recall as `grep`.
  The clearest statement of "the graph is a document you search," not a server.
- `understand-anything-plugin/agents/` — read `file-analyzer.md` then
  `article-analyzer.md` back-to-back to see the code path vs the knowledge path.
- `packages/core/src/fingerprint.ts` + `hooks/auto-update-prompt.md` — the
  incremental-freshness mechanism.

## Open questions for the study

- **Does grep-over-JSON scale?** For a 1M-LOC monorepo the graph JSON is large, and
  recall is `grep` + selective reads rather than an indexed query. Where does the
  "teach, don't query" bet break, and is that exactly where Graphify's MCP query
  surface starts to pay off?
- **How reliable is LLM extraction + alias repair vs deterministic parsing?** The
  alias maps are a confession that the LLM drifts. What's the residual error rate
  after normalization, and does `graph-reviewer` catch structural mistakes the
  alias maps can't (dangling edges, wrong direction)?
- **Is the `kind: "knowledge"` path the more interesting half?** The
  article/claim/source/cites/contradicts vocabulary points at a *research-notes /
  argument graph* that no other study entry models. For the Lossless corpus
  (context-v files, changelogs, prior decisions) this is arguably a closer fit than
  the code path — worth probing against our own Chroma corpus thesis.
- **Two graphs, one agent.** Graphify (MCP query) and Understand-Anything (grep +
  dashboard) could co-exist over the same repo. Is the right architecture
  *deterministic structure (Graphify) + LLM semantics (Understand-Anything)* fused,
  rather than choosing one?
