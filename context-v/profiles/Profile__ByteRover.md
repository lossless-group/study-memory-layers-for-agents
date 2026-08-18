---
name: ByteRover CLI Profile
slug: byterover
upstream: https://github.com/campfirein/byterover-cli
package: byterover-cli (npm)
license: Elastic-2.0 (source-available, not OSI-open)
maintainer: ByteRover (Campfire In) — Kevin Nguyen et al.
study: studies/memory-layers-for-agents
profile_path: studies/memory-layers-for-agents/byterover-cli
profile_kind: CLI + REPL + dashboard + hosted-platform (source-available)
date_created: 2026-05-26
site_uuid: 1a91d385-26bb-4f3b-b6b0-7b9a33e244fb
hex_code: 8mefyd
date_authored_initial_draft: 2026-05-26
date_authored_current_draft: 2026-05-26
lede: >-
  The memory store is a directory of Markdown you can cat, grep, and diff,
  with its own git history. Recall is BM25, not embeddings.
summary: >-
  Profile of ByteRover CLI as pinned in the memory-layers-for-agents study. It
  is the study's "developer is a first-class editor of the memory" entry — a
  CLI/REPL/dashboard trinity routed through one local daemon rather than a
  library an agent depends on. Covers the .brv/context-tree layout, the
  hierarchical _index.md summary nodes with their children-hash staleness key,
  the three-lane token-budget manifest, the runtime-signals sidecar kept out of
  version control on purpose, the four-phase curate write path with its HITL
  review gate, the "dream" prune/synthesize/consolidate eviction cycle, and the
  self-reported LoCoMo/LongMemEval numbers. Read it before any licensing
  decision that involves ByteRover — the Elastic-2.0 analysis is the section
  most likely to change a plan.
publish: true
---

# ByteRover CLI — Profile

A profile of ByteRover CLI as it lives in this study (`studies/memory-layers-for-agents/byterover-cli/`). ByteRover is the only entry in the study whose distribution shape is **a memory hub the developer uses, not a memory library an agent depends on**. Read alongside [[Profile__Volt]] (coding agent with built-in memory), [[Profile__Letta]] (git-backed block files), and [[Profile__Mem0]] (library + server + hosted under Apache-2.0) — those four bracket the design space for coding-adjacent memory.

## TL;DR

ByteRover CLI (`brv`) is an **interactive REPL + TUI + browser dashboard** that gives any coding agent persistent, version-controlled memory by *curating local Markdown files* into a hierarchical **context tree** rooted at `.brv/context-tree/` (`src/server/constants.ts:1, 28`). The CLI is the consumer the agent talks to; the agent talks back through 24 built-in tools (`src/agent/resources/tools/*.txt`) and the daemon writes results into `.md` files the developer can read, diff, and `vc commit`/`vc push` like git.

There is **no embedded vector DB**, **no SQLite memory store**, **no graph database**. Memory persists as a directory of curated topics + per-directory `_index.md` summary nodes + a `_manifest.json` token-budget allocator (`src/server/infra/context-tree/file-context-tree-manifest-service.ts:1-48`). Recall is **pure BM25** over Markdown (`src/server/infra/executor/search-executor.ts:1-42`) with optional LLM-synthesized answers on top via `brv query`. Runtime ranking signals (importance, maturity, access counts) live in a separate sidecar store (`src/server/core/domain/knowledge/runtime-signals-schema.ts:1-43`) so they don't dirty the git tree.

Distribution is a **single npm package + hosted cloud**. The local product is governed by **Elastic License 2.0** (`LICENSE:1, 9-11`): you may use, copy, and distribute, but you may not provide it to third parties as a hosted or managed service. The reader of this study should treat ByteRover as *prior art to read*, not a dependency to build on if a managed-service product is in the plan.

If you want one sentence: **ByteRover is a developer-facing memory hub — a CLI/REPL/dashboard trinity that turns a coding agent's accumulated knowledge into a versioned, human-readable Markdown context tree with optional cloud sync, distributed under Elastic 2.0.**

## Why this exists — the design bet

Most agent-memory tools assume the agent is the only editor and the human only sees memory through the agent's mouth. ByteRover bets the opposite: **the developer must be a first-class editor of the memory tree, and the agent is just one process curating into it**. That bet drives every architectural choice:

1. **Files-on-disk as ground truth.** Memory is `.md` files under `.brv/context-tree/` (`src/server/infra/context-tree/file-context-tree-service.ts:6, 55-59`). A developer can `cat`, `grep`, `git diff`, or open them in any editor. No SQLite required. No vector DB to inspect.
2. **Git-shaped version control for memory.** A full `brv vc` topic — `init/add/commit/log/branch/checkout/merge/clone/push/pull/fetch/remote/reset` (`README.md:155-173`) — implemented over `isomorphic-git` (`src/server/infra/git/isomorphic-git-service.ts`). The context tree is itself a git repo, separate from the project's source git repo.
3. **The dashboard, the REPL, and the agent share one daemon.** Single process at `127.0.0.1` (`src/server/constants.ts:43`), Socket.IO transport, agent forked as a child per project. Human edits via `brv webui` and agent curates via the REPL flow into the same store.
4. **LLM-curated hierarchy, not LLM-extracted facts.** The agent is asked to call `curate` with structured operations (ADD/UPDATE/MERGE/DELETE) at chosen paths (`src/agent/resources/tools/curate.txt:30-77`). Hierarchy is intentional; there is no extraction prompt that yanks "preferences" out of arbitrary chat.
5. **Local-first, cloud-optional.** Everything works without `brv login`. Cloud (ByteRover Cloud, app.byterover.dev) adds team sync, multi-machine, hosted LLM credits, SOC 2 (`README.md:105-124`) — but it is purely additive.

## Storage topology

| Layer | Backend | Purpose | Code |
|---|---|---|---|
| Curated knowledge | `.brv/context-tree/<domain>/<topic>/<file>.md` | LLM-curated topic files, written by `curate` | `src/server/infra/context-tree/file-context-tree-service.ts:52-62` |
| Hierarchical summaries | `_index.md` per directory, with `condensation_order` per depth | LLM-summarized child nodes, three-tier escalation w/ deterministic fallback | `src/server/infra/context-tree/file-context-tree-summary-service.ts:84-157` |
| Manifest | `_manifest.json` (sibling of root context-tree) | Three-lane (summaries/contexts/stubs) token budget allocator | `src/server/infra/context-tree/file-context-tree-manifest-service.ts:1-48` |
| Runtime signals sidecar | `IKeyStorage` keyed by `["signals", ...pathSegments]` (per-machine) | importance, maturity, recency, accessCount, updateCount — never in markdown frontmatter | `src/server/core/domain/knowledge/runtime-signals-schema.ts:17-35` |
| Agent scratch memory | Blob storage, key `memory-<nanoid12>`, JSON-serialized | Per-session scratch pad written by `write_memory` tool | `src/agent/infra/memory/memory-manager.ts:127-135, 562-579` |
| Search index | minisearch BM25 (in-memory, rebuilt) | Pure-retrieval engine behind `brv search`, no LLM | `src/server/infra/executor/search-executor.ts:1-42` |
| VC | `isomorphic-git` over `.brv/context-tree/.git/` | Branch/commit/push/pull for the context tree | `src/server/infra/git/isomorphic-git-service.ts` |
| Cloud (optional) | ByteRover Cloud (managed service) | Team sync, multi-machine, hosted LLM credits, SOC 2 | `README.md:105-124` |
| Archive stubs | `.archive.md` with `original_path` / `evicted_at` / `points_to` frontmatter | Tombstones for pruned files; preserve references | `src/server/infra/context-tree/summary-frontmatter.ts:65-80` |

The load-bearing idea: **the markdown tree is the database, the manifest is the index, the sidecar is the access log, and git is the audit trail**. Nothing else is required.

## Schema of a curated topic

The `curate` tool's two-part content model (`src/agent/resources/tools/curate.txt:1-23`):

```
tags:        ["authentication", "security", "jwt"]
keywords:    ["jwt", "refresh_token", "rotation"]
rawConcept:
  task:      "What is the task or subject"
  changes:   ["code changes, process updates, decisions"]
  files:     ["services/auth.go", ...]
  flow:      "step1 -> step2 -> step3"
  timestamp: "2025-03-18"
narrative:
  structure:    "file layout / process hierarchy / timeline"
  dependencies: "prerequisites, blockers, relationship info"
  highlights:   "key capabilities, deliverables, outcomes"
facts:       [{statement, category, subject, value}, ...]
relations:   ["@domain/other_topic"]
```

Per-operation metadata required for every call: `reason` (why), `summary` (one-line for the human reviewer), `confidence: high|low`, `impact: high|low` (`src/agent/resources/tools/curate.txt:24-29`). The HITL review queue (`brv review pending|approve|reject`) gates writes when enabled.

Summary node frontmatter (`src/server/infra/context-tree/summary-frontmatter.ts:42-58`):

```yaml
type: summary
condensation_order: 0..N        # depth in the tree
summary_level: d0|d1|d2|...
covers: [child1.md, child2.md]
covers_token_total: 4200
token_count: 850
compression_ratio: 0.20
children_hash: <sha256 of sorted child contentHashes>
```

`children_hash` is the staleness key: if it differs from the live hash, the summary is regenerated (`src/server/infra/context-tree/file-context-tree-summary-service.ts:54-82`).

Runtime signals (sidecar only, not in markdown — explicit design choice, `runtime-signals-schema.ts:1-9`):

```
accessCount, updateCount, recency [0,1], importance [0,100], maturity: core|draft|validated
```

These change on every query and would otherwise dirty VC state across teammates.

## Write policy — agent-curated, gated, propagated

`brv curate` runs in **four phases** (`CLAUDE.md` architecture notes, `src/server/infra/executor/curate-executor.ts`):

1. **Foreground 1–3**: LLM proposes ADD/UPDATE/MERGE/DELETE operations under `domain/topic/subtopic` paths (snake_case auto-applied, `src/agent/resources/tools/curate.txt:75-76`); operations write Markdown files; failed ops are recorded in the curate log (`src/server/infra/storage/file-curate-log-store.ts:13-48`).
2. **HITL gate** (when `brv review` enabled): each operation marked `needsReview: true` is held in `file-review-backup-store.ts` until `brv review approve <id>` or `brv review reject <id>`. Reject restores the pre-operation snapshot.
3. **Detached Phase 4**: summary regeneration + manifest rebuild are pushed to the daemon's `PostWorkRegistry`, which serializes per-project to prevent concurrent `_index.md` writes. Coordinates with `dream-lock-service.ts`.
4. **Staleness propagation**: `propagateStaleness(changedPaths, …)` walks ancestors and regenerates any `_index.md` whose `children_hash` no longer matches (`file-context-tree-summary-service.ts:172-178`).

No append-only log, no automatic supersession, no embedding step. The "merge" of related knowledge is explicit: the agent must emit a `MERGE` operation with `mergeTarget` (`curate.txt:67-69`).

## Recall API — three layers

1. **`brv search <query>`** — pure BM25, no LLM, no token cost (`search-executor.ts:1-9`). Returns ranked results with `path`, `score`, `title`, `excerpt`, and `origin: local|shared` (`src/agent/resources/tools/search_knowledge.txt:14-21`). This is the engine agents call via the `search_knowledge` tool.
2. **`brv query <question>`** — Tier-0-to-4 retrieval with LLM synthesis on top (`src/server/infra/executor/query-executor.ts`). The manifest's three lanes (summaries / contexts / stubs) feed a token-budgeted context-injection step. Cached via `query-result-cache.ts` keyed by similarity (`query-similarity.ts`).
3. **`brv swarm query`** — multi-provider memory federation across pluggable adapters (byterover, gbrain, local-markdown, memory-wiki, obsidian) with RRF fusion (`src/agent/infra/swarm/`). The swarm layer lets the agent treat several backends as a single recall surface.

Knowledge sources (`brv source add <path>`) let one project's agent read another project's context tree, with a local-score boost (`SHARED_SOURCE_LOCAL_SCORE_BOOST = 0.1`, `src/server/constants.ts:12`) and write isolation — shared sources are read-only.

## Eviction & compaction — the "dream" cycle

ByteRover has a real eviction story, unlike most entries in this study. `brv dream` runs background consolidation (`src/server/infra/dream/operations/`):

- **prune.ts** finds candidates two ways: (a) archive service importance decay (draft files with importance < 35), (b) mtime staleness (draft: 60 days, validated: 120 days, **core: never**). Caps at 20 stalest. One LLM call decides ARCHIVE / KEEP / MERGE_INTO per candidate. Archived files become **archive stubs** with frontmatter pointing to the new location (`prune.ts:1-13, 23-30`).
- **synthesize.ts** detects cross-domain patterns from domain `_index.md` files and writes new synthesis files as draft context entries; BM25 deduplicates against existing files at threshold 0.5 (`synthesize.ts:1-13, 57`).
- **consolidate.ts** — third operation in the dream cycle.

`brv dream --undo` rolls back, `--detach` runs background, locked per-project via `dream-lock-service.ts`. Maturity tiers (`core|draft|validated`, `runtime-signals-schema.ts:27`) govern what's protected: core never evicts, draft evicts aggressively.

## Scopes & namespacing

ByteRover's scope is **the project directory**, not a `user_id`. A project is anything with `.brv/config.json` (`PROJECT_CONFIG_FILE`, `src/server/constants.ts:3`). The canonical project resolver (`server/infra/project/`) walks the priority `flag > direct > linked > walked-up > null`.

- **Worktrees** (`brv worktree add`) — `.brv/` becomes a *pointer file* to a parent project (`WORKTREES_DIR = 'worktrees'`, `WORKTREE_LINK_METADATA = 'link.json'`, `src/server/constants.ts:7-8`); a git-style move that lets a subdirectory share its parent's context tree.
- **Spaces** (ByteRover Cloud) — team-level grouping of projects; cloned via `brv vc clone`.
- **Knowledge sources** (`brv source add`) — read-only links to another project's context tree.
- **Agent scratch memory** — per-agent-session, never persisted to the context tree; written via `write_memory`, scoped by `memory-<nanoid12>` blob keys (`memory-manager.ts:128-129, 562-565`).

Notable absence: **no `user_id` axis**. ByteRover assumes the project directory is the unit of memory, and team-level sharing is the cloud's job.

## Operational story

Three artifacts ship from one package:

1. **CLI** (`brv` subcommands) — oclif v4-built, ~30 top-level commands + topic groups (`vc`, `hub`, `worktree`, `source`, `space`, `review`, `connectors`, `curate`, `model`, `providers`, `swarm`, `query-log`, `settings`) per `README.md:131-203`.
2. **REPL + TUI** — React/Ink interactive surface started by bare `brv` (`src/tui/repl-startup.tsx`); slash commands map to the same daemon transport events. Esc cancels streaming.
3. **Web dashboard** — `brv webui` (default port 7700, `src/server/constants.ts:63`) — Vite-built React app, connects to the daemon over Socket.IO. Eight pages: home, changes, configuration, contexts, tasks, analytics, project-selector, not-found.

All three are daemon-routed (`server/infra/daemon/`), and `tui/` is forbidden by ESLint from importing `server/`, `agent/`, or `oclif/` — clean transport-event boundary.

**Integration surface** — ByteRover ships an MCP server (`brv mcp`, `src/server/infra/mcp/mcp-server.ts`) with two tools: `brv-query-tool` and `brv-curate-tool` (`src/server/infra/mcp/tools/`). Cursor, Claude Code, Windsurf, Cline, Codex, OpenCode, Amp, and ~15 others wire in by registering ByteRover as an MCP server; rules files (e.g. `AGENTS.md` shared by Amp/Codex/OpenCode) carry an agent-name footer between `<!-- BEGIN/END BYTEROVER RULES -->` markers (`src/server/infra/connectors/shared/constants.ts`).

**LLM providers**: 20 listed (`README.md:285-307`) — Anthropic, OpenAI, Google, Groq, Mistral, xAI, Cerebras, Cohere, DeepInfra, DeepSeek, OpenRouter, Perplexity, TogetherAI, Vercel AI SDK, Minimax, Moonshot, GLM, GLM Coding Plan, OpenAI-Compatible, plus ByteRover's own hosted models. Per-provider OAuth or API-key via `brv providers connect`.

**Settings** persist at `<BRV_DATA_DIR>/settings.json` (`README.md:237-241`, default `~/Library/Application Support/brv/settings.json` on macOS). Keys include `agentPool.maxSize=10`, `agentPool.maxConcurrentTasksPerProject=5`, `llm.iterationBudgetMs=600000`, `llm.requestTimeoutMs=120000`, `taskHistory.maxEntries=1000`. Most require `brv restart`.

## Headline benchmark numbers

Source: `paper/README.md:7-18`, also surfaced at `README.md:46-63`.

| Benchmark | Result | Scope |
|---|---|---|
| LoCoMo | **96.1%** | 1,982 questions, 272 docs, ultra-long conversations (~20K tokens, 35 sessions) |
| LongMemEval-S | **92.8%** | 500 questions, 23,867 docs, ~48 sessions per question, 6 memory abilities |

Both LLM-as-Judge accuracy. Run from the production `byterover-cli` codebase, **not a research prototype** (`paper/README.md:5`). Paper claims to be at `arxiv.org/abs/2604.01599` — that URL is a placeholder (2604 is a future year); the PDF is self-hosted at `byterover.dev/paper`. Treat the numbers with the same independent-verification caveat as [[Profile__Mem0]]'s headline scores, and see [[Profile__StateBench]] for why the study cares.

## What's inside this submodule

| Path | What's there |
|---|---|
| `src/agent/` | Agent infrastructure: 24 tools (`resources/tools/*.txt`), 20 LLM providers (`infra/llm/`), agentic map (`infra/map/`), scratch memory manager (`infra/memory/`), swarm federation (`infra/swarm/`) |
| `src/server/` | Daemon: context-tree services (read/write/summary/manifest/snapshot), `dream/` operations (prune/synthesize/consolidate), `vc/` + `git/`, `executor/` (curate/query/search/folder-pack/dream), `mcp/` server, `connectors/` (rules/skill/mcp/hook/shared), `billing/`, `storage/` (file-based stores) |
| `src/tui/` | React/Ink REPL, 24 feature modules |
| `src/webui/` | Vite + React browser dashboard, 16 feature panels, 8 pages |
| `src/oclif/` | CLI command surface — `~30` top-level + topic groups |
| `src/shared/` | Cross-module: constants, types, transport events |
| `bin/run.js`, `bin/dev.js` | Entry points; `kill-daemon.js` for dev resets |
| `paper/` | LaTeX source for the ByteRover paper (~1,300 lines `main.tex`, 32 refs `references.bib`); `build.sh` produces a PDF locally |
| `packages/byterover-packages/` | Shared UI components submodule (published as `@campfirein/byterover-packages`) |
| `scripts/` | `install.sh`, `uninstall.sh`, `openclaw-setup.sh`, `byterover-legacy-plugin.sh`, frontmatter migrator |

If you only read one file: `src/server/infra/context-tree/file-context-tree-summary-service.ts` — it shows the whole hierarchical-summary story (staleness, three-tier escalation, deterministic fallback, children-hash invariant) in one place.

## Mental model for using it well

- **Treat memory as files first.** If you can't `cat` it, ByteRover isn't doing its job. The whole design assumes the developer reads the tree directly.
- **Let the agent curate, not extract.** Mem0-style "extract facts from arbitrary chat" is not ByteRover's shape. Ask the agent to `curate` deliberately with `reason` + `summary` so the human reviewer can follow.
- **Use `brv review` for anything touching shared knowledge.** The HITL queue is the difference between "agent broke our context tree overnight" and "I see three pending operations to approve."
- **Don't fight the file layout.** Domains are LLM-chosen at curate time (`curate.txt:78-84`); the agent picks `auth/`, `caching/`, `data_models/` etc. Override by editing the markdown directly — the next curate run respects what's there.
- **Pair with [[Profile__Volt]] only carefully.** Volt is its own agent with its own Postgres-backed log; ByteRover is an MCP/integration target. They overlap on "coding-agent memory" but the seams are different.
- **The dream cycle is opt-in.** `brv dream` runs eviction/synthesis when invoked; nothing background-evicts by default. For long-running projects, schedule it.

## When NOT to reach for this

- **You need a memory layer you can productize as a managed service.** Elastic 2.0 (`LICENSE:9-11`) explicitly forbids providing the software to third parties as a hosted/managed service. The CLI is fine to embed in your own developer workflow; reselling its features as a SaaS is not.
- **You want a vector-DB-backed semantic memory.** ByteRover's recall is BM25 + LLM-synth, not embeddings. If "fuzzy semantic recall on raw chat history" is the requirement, see [[Profile__Mem0]] (Apache-2.0, vector + entity store) or [[Profile__Graphiti]].
- **You want supersession-as-first-class.** ByteRover's MERGE is explicit and human-reviewable, not automatic; if the failure mode is "old facts resurrect," see [[Profile__Neo]] or [[Profile__StateBench]] for the discipline.
- **You don't want a daemon.** ByteRover runs a background daemon at `127.0.0.1:dynamic` (`src/server/constants.ts:43, 58-61`) with per-project agent child processes. For low-overhead use (CI, one-shot scripts), this is heavier than a library import.
- **Your "memory" is project documentation a human writes.** Then you want the Lossless `context-vigilance` skill (markdown + wikilinks + git, no daemon) — same intuition but without the agent-curated overhead.

## How this compares to the rest of the study

| Axis | ByteRover | Mem0 | Volt | Letta |
|---|---|---|---|---|
| **Shape** | CLI + REPL + dashboard + MCP target | Library / server / hosted | Coding agent w/ embedded memory | Stateful-agent runtime (server) |
| **Storage** | Markdown context tree + sidecar + git | Vector + entity + SQLite history | Postgres DAG | Postgres + optional git-backed blocks |
| **Write policy** | LLM-curated ADD/UPDATE/MERGE/DELETE, HITL-gated | Append-only + entity link (v3 single-pass) | Verbatim append + async summary | Block-level explicit edits |
| **Recall** | BM25 + LLM-synth + manifest lanes | Hybrid (semantic + BM25 + entity) | Bindle expansion + grep | Block injection + archival search |
| **Eviction** | `dream` cycle: prune/synthesize/consolidate, maturity-tiered | None in OSS | Dolt evicts bindles | Manual + summarization |
| **Scope axis** | Project directory + worktree + shared source | user/agent/run/actor (required) | Conversation | Agent ID |
| **License** | **Elastic-2.0** (source-available, no third-party hosting) | Apache-2.0 | GPL-3.0 (per its profile) | Apache-2.0 |
| **Human-editable storage** | Yes — markdown + git | No (vector + SQLite) | Partial (SQL queryable) | Partial (with git-backed blocks) |
| **Daemon required** | Yes | No (library mode) | No (process-local) | Yes (server) |
| **Best fit** | Developer running multiple coding agents who wants one shared, versioned memory hub | Any agent needing cross-session user memory | Long-horizon coding sessions inside the Volt agent | Long-lived agents needing block-level editable state |

## License — Elastic 2.0, in detail

ByteRover ships under **Elastic License 2.0** (`LICENSE:1, 9-15`), a **source-available, non-OSI license**. The salient terms:

- *You may*: use, copy, distribute, modify, and prepare derivative works for your own use.
- *You may not*: "provide the software to third parties as a hosted or managed service, where the service provides users with access to any substantial set of the features or functionality of the software" (`LICENSE:10`).
- *You may not*: disable or circumvent license-key functionality (`LICENSE:12`).
- *You may not*: remove copyright/licensing notices (`LICENSE:14`).

For this study, that means: **read the code, learn from the architecture, treat patterns as prior art**. Building an internal tool that depends on `byterover-cli` is fine. Re-hosting ByteRover's features as a competing memory SaaS — say, by stripping the cloud dependency and exposing the daemon to external customers — is not. Contrast with [[Profile__Mem0]] (Apache-2.0, no such restriction) and [[Profile__Letta]] (Apache-2.0) — ByteRover's licensing is the sharpest in the study.

## One-line summary

> ByteRover CLI is a developer-facing memory hub — CLI + REPL + browser dashboard, all routed through one local daemon — that stores agent-curated knowledge as a versioned Markdown context tree with manifest-budgeted BM25 recall, a "dream" eviction cycle, and optional cloud sync, distributed under Elastic 2.0 so it's source-available for reading and embedding but off-limits as a hosted-service competitor.
