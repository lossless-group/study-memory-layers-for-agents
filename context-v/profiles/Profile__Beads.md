---
name: Beads Profile
slug: beads
upstream: https://github.com/gastownhall/beads
package: "@beads/bd (npm), beads-mcp (PyPI), beads (Homebrew)"
license: MIT
maintainer: gastownhall org — Go module path github.com/steveyegge/beads (Steve Yegge et al.)
study: studies/memory-layers-for-agents
profile_path: studies/memory-layers-for-agents/beads
profile_kind: CLI + embedded/server SQL database + MCP server
date_created: 2026-05-28
---

# Beads — Profile

A profile of Beads as it lives in this study (`studies/memory-layers-for-agents/beads/`). Cites pinned paths so you can jump to source rather than trust paraphrase. Read alongside [[Profile__Volt]] and [[Profile__ByteRover]] — the three coding-agent-memory entries — but note that Beads makes a categorically different bet from *every* entry in the study: it is not a recall layer over conversation at all. It is a **versioned, dependency-typed issue tracker repurposed as an agent's working memory** — the agent's to-do graph, not its episodic past.

## TL;DR

Beads (`bd`) is a **distributed, graph-based issue tracker for AI coding agents**, written in Go, storing everything in **[Dolt](https://github.com/dolthub/dolt)** — a Git-like, version-controlled, MySQL-compatible SQL database. The pitch (`README.md:15`): *replace messy markdown TODO plans with a dependency-aware graph so agents can handle long-horizon tasks without losing context across compaction, session resets, or account rotations.*

There is **no vector store, no embeddings, no semantic similarity anywhere in the recall path.** Recall is SQL plus graph traversal: `bd ready` lists tasks with no open blockers (backed by the `ready_issues` SQL view, `internal/storage/schema/migrations/0017_create_ready_issues_view.up.sql`), `bd show <id>` returns a task and its audit trail, `bd dep` walks the typed dependency edges. The "memory" Beads gives an agent is **procedural** — what's done, what's blocked, what's next — not episodic or factual.

Three load-bearing design moves:

1. **Hash-based IDs** (`bd-a1b2`) — SHA256 of `title|description|creator|timestamp|nonce` → base36, 3–8 chars (`internal/idgen/hash.go:55-85`). Zero merge collisions across parallel agents and git branches, because there is no shared auto-increment counter.
2. **Dolt as the substrate** — cell-level merge, native branching, commit history on every write, and built-in peer sync (`bd dolt push`/`bd dolt pull` against `refs/dolt/data`). The issue graph is itself a versioned database you can branch and merge like code.
3. **AI memory-decay compaction** — closed issues are summarized by **Claude Haiku** (`internal/compact/haiku.go`) into Summary / Key Decisions / Resolution, with the full original preserved in snapshot tables for recovery.

A separate, literal "memory" surface — `bd remember` / `recall` / `memories` / `forget` (`cmd/bd/memory.go`) — stores slugified key→value insights in the config KV table and re-injects them into every session via `bd prime` (`cmd/bd/prime.go`). This is the closest thing in Beads to "agent memory" in the sense the rest of the study means it, and it is deliberately dumb: substring search, no embedding, no extraction.

**One sentence:** *Beads is structured working memory for coding agents — a typed, dependency-aware task graph in a Git-like SQL database (Dolt), with hash IDs for conflict-free multi-agent writes, an immutable event audit trail, Haiku-summarized memory decay for closed work, and a small key-value insight store re-primed into every session.*

## Why this exists — the design bet

Every other entry in the study optimizes **recall over the past**: "what did the user say," "what facts do we know," "what happened in session 14." Beads optimizes **state over the present and future**: "what am I supposed to be doing, what's blocking it, what's already done, and how do I not redo it after my context window gets compacted."

The motivating failure (`README.md:15`, `AGENTS.md`): coding agents track work in throwaway markdown TODO lists that (a) don't survive context compaction, (b) don't model dependencies, (c) collide when multiple agents or branches edit them, and (d) silently lose the thread of long-horizon tasks. Beads' answer is to treat the task list as a **first-class, versioned, queryable graph** the agent reads and writes through a CLI, with the database — not the chat transcript — as the source of truth.

That makes Beads the study's clearest example of **procedural / task memory** as opposed to **episodic / semantic memory**. It belongs in the study precisely because it forces the question the recall-focused entries quietly skip: *is "agent memory" even one thing?* Beads says the durable, structured part of what an agent needs to remember is its own work graph, and that part is better served by an issue tracker than by a RAG pipeline.

## Storage topology — Dolt, two modes, no vector anything

The whole memory layer is a Dolt database. Dolt is MySQL-wire-compatible but Git-shaped: tables have commit history, branches, and three-way cell-level merge. Two deployment modes (`README.md:107-151`):

| Mode | Engine | Data dir | Writers | Code |
|---|---|---|---|---|
| **Embedded** (default, `bd init`) | Dolt in-process | `.beads/embeddeddolt/` | Single (file-lock enforced) | `internal/storage/embeddeddolt/` |
| **Server** (`bd init --server`) | External `dolt sql-server` (TCP `:3307` or Unix socket) | `.beads/dolt/` | Multiple concurrent | `internal/storage/dolt/`, `internal/doltserver/` |

There is no second backend. A grep for `sqlite` in `internal/storage/` returns nothing tracked — `BENCHMARKS.md`'s references to `SQLiteStorage` are stale naming from an earlier engine; the live storage tree is `internal/storage/dolt` + `internal/storage/embeddeddolt`, both Dolt. `FEDERATION-SETUP.md` confirms it explicitly: *"Federation requires the Dolt storage backend (the only supported backend)."*

This is the **single sharpest contrast in the study**: where Mem0, MemPalace, Supermemory, Hindsight, and RetainDB all build on embeddings + vector search, Beads has none. Recall correctness here is a SQL-query-and-graph-traversal problem, not a nearest-neighbor problem.

> Coincidence worth flagging so you don't trip on it: [[Profile__Volt]] names one of its *eviction modes* "Dolt." Beads *is built on* Dolt the database. Unrelated — but if you read the two profiles back to back, the word means two different things.

## Schema of a memory — the `issues` table is ~50 columns wide

The canonical record is a row in `issues` (`internal/storage/schema/migrations/0001_create_issues.up.sql`), modeled in Go as `types.Issue` (`internal/types/types.go:16-105`). The Go struct is the cleanest read; the column groups:

- **Identity:** `id` (hash ID, PK), `content_hash` (SHA256 of canonical content, internal-only `json:"-"`).
- **Content:** `title`, `description`, `design`, `acceptance_criteria`, `notes`, `spec_id` — five structured prose fields, not one blob. Compaction targets exactly these (see below).
- **Workflow:** `status`, `priority` (int, `0` = P0/critical, no omitempty), `issue_type`.
- **Assignment:** `assignee`, `owner` (human git-author email for attribution), `estimated_minutes`.
- **Timestamps:** `created_at`, `created_by`, `updated_at` (`ON UPDATE CURRENT_TIMESTAMP`), `started_at`, `closed_at`, `close_reason`, `closed_by_session` (the Claude Code session that closed it), plus scheduling `due_at` / `defer_until`.
- **External integration:** `external_ref` (`gh-9`, `jira-ABC`), `source_system` (federation adapter).
- **Extension:** `metadata JSON` — arbitrary well-formed JSON, validated on write.
- **Compaction:** `compaction_level`, `compacted_at`, `compacted_at_commit` (git hash at compaction time), `original_size`.
- **Messaging / ephemerality:** `sender`, `ephemeral`, `wisp_type`, `pinned`, `is_template`.
- **Coordination primitives:** `await_type`/`await_id`/`timeout_ns`/`waiters` (async gates), `hook_bead`/`role_bead`/`agent_state`/`rig` (swarm/agent orchestration).

### Status and type taxonomies are enumerated, not free-form

`Status` (`types.go:327-333`): `open`, `in_progress`, `blocked`, `deferred`, `closed`, `pinned`, `hooked`. Custom statuses are allowed and carry a `StatusCategory` (`active` / `wip` / `done` / `frozen`) so views and `bd ready` know how to treat them (`types.go:374-402`).

`IssueType` (`types.go:518-546`): `bug`, `feature`, `task`, `epic`, `chore`, `decision`, `message`, `molecule`, `gate`, `spike`, `story`, `milestone`, plus custom types. Notable that **`decision` is a first-class issue type** — closer to a knowledge artifact than a work item.

### The dependency graph is a rich typed edge set

`dependencies` (`0002_create_dependencies.up.sql`) is the graph. Edge `type` is one of ~19 well-known `DependencyType` values (`types.go:776-812`), grouped by what they do:

- **Workflow (affect ready-work calculation):** `blocks`, `parent-child`, `conditional-blocks` (B runs only if A fails), `waits-for` (fan-out gate).
- **Association:** `related`, `discovered-from`.
- **Knowledge-graph links:** `replies-to` (conversation threading), `relates-to` (loose edges), `duplicates` (dedup), `supersedes` (**version chain** — the same supersession primitive [[Profile__Neo]] and [[Profile__RetainDB]] make first-class, here expressed as a graph edge).
- **Entity links:** `authored-by`, `assigned-to`, `approved-by`, `attests` (X attests Y has skill Z).
- **Reference:** `tracks`, `until`, `caused-by`, `validates`, `delegated-from`.

So Beads *is* a typed knowledge graph — just one whose nodes are work items and whose edges encode workflow, provenance, and supersession rather than world facts.

### Supporting tables

- **`events`** (`0005`, UUID PK) — immutable audit trail: `event_type`, `actor`, `old_value`, `new_value`, `comment`, `created_at`. This is the append-only history layer; the `issues` row is the mutable current state.
- **`comments`**, **`labels`** — standard issue-tracker satellites.
- **`config`** / **`metadata`** — KV stores; `config` is where `bd remember` lives (key `kv.memory.<slug>`).
- **`issue_snapshots`** (`0009`) + **`compaction_snapshots`** (`0010`) — pre-compaction originals (full content + archived events) so memory decay is reversible.
- **`interactions`** (`0014`) — an **LLM-call audit log**: `kind`, `actor`, `issue_id`, `model`, `prompt`, `response`, `error`, `tool_name`, `exit_code`, `parent_id`. Structurally similar to the Lossless `claude-code-tool-traces` collection — agent telemetry as a queryable table.
- **`federation_peers`** (`0015`) — peer name, remote URL, encrypted credentials, and a `sovereignty` tier (see operational story).
- **`wisps`** (`0020`) — a *parallel, dolt-ignored* copy of the `issues` schema for **ephemeral local beads** (messages, transient agent state). Not synced via git; subject to TTL-based compaction by `wisp_type`. The `no_history` flag (`types.go:80`, migration `0023`) keeps a wisp out of GC.
- **Views** `ready_issues` (`0017`) and `blocked_issues` (`0018`) — the ready/blocked computation is a SQL view, not application code, which is why `bd ready` is fast and consistent.

Migration `0037_uuid_primary_keys.up.sql` is worth reading on its own: it converts the few remaining `BIGINT AUTO_INCREMENT` PKs to UUID, with the rationale stated in the file header — *"independent AUTO_INCREMENT counters across federated clones produce conflicting IDs on push/pull."* The same conflict-avoidance instinct that produced hash IDs for issues.

## ID generation — hash-based, hierarchical, conflict-free

`internal/idgen/hash.go:55-85`: `GenerateHashID` builds `title|description|creator|timestamp.UnixNano()|nonce`, SHA256s it, and base36-encodes the first N bytes to a 3–8 char suffix → `bd-a1b2`. The `nonce` parameter handles the rare hash collision. Base36 (0-9a-z) over hex for density (`hash.go:53`).

Hierarchy is in the ID string itself (`README.md:71-78`): `bd-a3f8` (epic) → `bd-a3f8.1` (task) → `bd-a3f8.1.1` (subtask). `internal/storage` reserves child counters per parent (`child_counter_reservation.go`, migration `0008`).

The payoff (`README.md:54`): two agents on two branches both create issues, both push, and there is **no ID collision to merge** — because IDs are content-hashed, not sequence-allocated. This is the multi-agent-write story the conversation-memory entries don't have to solve.

## Write policy — mutate-with-audit, atomic claims, Dolt-versioned

Writes mutate the `issues` row and append to `events`. `bd update <id> --claim` is **atomic** (sets `assignee` + `status=in_progress` in one operation, `README.md:65`) so two agents can't both claim the same task. Because the store is Dolt, every write is a versioned change with full commit history and three-way cell-level merge — concurrent edits to *different fields of the same issue* merge cleanly rather than conflicting.

Supersession is explicit and graph-shaped: emit a `supersedes` dependency edge (`types.go:794`) to chain an issue to the one it replaces. Contrast [[Profile__RetainDB]] (bi-temporal `validFrom`/`validUntil` columns) and [[Profile__Neo]] (`superseded_by` chain on a fact) — Beads encodes the same idea as a typed edge in the work graph.

There is no LLM in the write path for ordinary task CRUD. The only LLM call Beads makes on its own is **compaction** (below).

## The literal memory surface — `bd remember` / `recall` / `prime`

This is the part of Beads that maps onto "agent memory" as the rest of the study uses the term, and it is deliberately minimal (`cmd/bd/memory.go`):

- **`bd remember "<insight>"`** — stores the insight under config key `kv.memory.<slug>`, where `<slug>` is the first ~8 words slugified (`memory.go:21-42`), or an explicit `--key`. Re-using a key updates in place. Stored as a plain string in the `config` table.
- **`bd recall <key>`** — fetch one memory by key.
- **`bd memories [search]`** — list all, or **substring-filter** by key/value (`memory.go:146-155`). No embedding, no ranking.
- **`bd forget <key>`** — delete.

The recall trigger is `bd prime` (`cmd/bd/prime.go`): designed to run as a **SessionStart hook** for Claude Code / Codex / Gemini CLI, it prints workflow context plus all stored memories so the agent re-acquires them after every context compaction (`prime.go:55-72`). `--memories-only` injects just the memories. A project can override the whole output with `.beads/PRIME.md` (`prime.go:70`).

So the mechanism for "memory that survives compaction and account rotation" (`memory.go:48`) is: **write small insights as KV, re-inject them every session via a hook.** It is the opposite philosophy from [[Profile__Letta]]'s self-editing core memory or [[Profile__Mem0]]'s extraction pipeline — there is no automatic capture; the agent (or human) decides what to `remember`.

## Recall surface — SQL views and graph queries, JSON for agents

| Command | What it returns | Backing |
|---|---|---|
| `bd ready` | Tasks with no open blockers (auto-ready detection) | `ready_issues` view (`0017`) |
| `bd blocked` | Tasks waiting on open dependencies | `blocked_issues` view (`0018`) |
| `bd show <id>` | One issue + dependencies + comments + event audit trail | `issues` + `dependencies` + `events` |
| `bd list` / search | Filter by status/priority/type/label | `issues` indexes |
| `bd dep add/tree` | Walk the typed dependency graph | `dependencies` |
| `bd prime` | Workflow context + persistent memories | `config` KV |
| `bd memories [q]` | List/substring-search KV memories | `config` KV |

Every command supports `--json` for agent consumption (`README.md:53`). The recall primitive is "query the graph," not "embed the query and find neighbors." Cycle detection over the dependency graph is a benchmarked hot path (`BENCHMARKS.md:30-34`), as is `GetReadyWork` over 10K–20K-issue datasets.

## Eviction & compaction — AI "memory decay" with reversible snapshots

Beads has a real, AI-driven compaction story (`internal/compact/`), framed as *"semantic memory decay"* (`README.md:55`):

- **Trigger / eligibility:** `compactableStore.CheckEligibility(issueID, tier)` (`compactor.go:35-41`) gates which closed issues are candidates; `GetTier1Candidates` / `GetTier2Candidates` are benchmarked (`BENCHMARKS.md:26-28`).
- **Summarizer:** Claude Haiku via the official `anthropic-sdk-go` (`haiku.go:15, 47-74`), model from `config.DefaultAIModel()`, `MaxTokens: 1024`, with exponential-backoff retry on 429/5xx (`haiku.go:127-227`). API key from `ANTHROPIC_API_KEY` → `ai.api_key` config; absent a key, compaction silently degrades to dry-run (`compactor.go:64-72`).
- **Tier 1 transform** (`compactor.go:87-157`): replace `description` with a structured Summary / Key Decisions / Resolution (prompt at `haiku.go:264-291`), and blank `design` / `notes` / `acceptance_criteria`. **Refuses to compact if the summary isn't actually shorter** (`compactor.go:122-130`).
- **Reversibility:** the original content + archived events are written to `issue_snapshots` / `compaction_snapshots` (`0009`, `0010`) before the row is overwritten; `compaction_level`, `original_size`, and the git commit hash at compaction time are recorded on the issue.
- **Batching:** `CompactTier1Batch` runs with a concurrency semaphore (default 5, `compactor.go:13, 168-212`).
- **Wisp TTL:** ephemeral `wisps` are compacted/expired on a TTL keyed by `wisp_type`, distinct from the issue-summarization path.

This is the most concrete eviction policy in the study after [[Profile__ByteRover]]'s "dream" cycle — and unlike most entries, it keeps the original recoverable rather than discarding it.

## Serialization on disk — Dolt is truth, JSONL is the export

The persisted form is the Dolt database under `.beads/`. For interchange there is also **`.beads/issues.jsonl`** — newline-delimited JSON, one issue per line. The repo's own dogfooded copy (`issues.jsonl` at root) shows the shape:

```json
{"id":"bd-main-idj","title":"Pattern-collapse pass: mechanical cruft inventory and reduction",
 "description":"...","status":"in_progress","priority":2,"issue_type":"chore",
 "owner":"maphew@gmail.com","created_at":"2026-04-18T16:19:12Z","created_by":"matt wilkie",
 "updated_at":"2026-04-18T16:30:16Z","started_at":"2026-04-18T16:30:16Z",
 "dependency_count":0,"dependent_count":0,"comment_count":0}
```

Crucially, the README is emphatic that **JSONL is not the source of truth** (`README.md:122-126, 170-173`): it does not capture Dolt branches, commit history, working-set state, or non-issue tables. Cross-machine sync is `bd dolt push`/`bd dolt pull` against `refs/dolt/data` — *not* git-committing the JSONL. For a restorable backup you use `bd backup` or a Dolt backup, not the export.

The study's "could a non-AI program parse it cleanly?" check: **yes, trivially** — the JSONL export is plain JSON. But "is the export the database?" — **no**. That gap (legible export vs. authoritative versioned DB) is itself the interesting design point, and the inverse of [[Profile__ByteRover]] / [[Profile__Letta]]-git-mode, where the human-readable files *are* the store.

## Scopes & namespacing

Beads' scope is the **project directory** (`.beads/`), discovered by walking up from the cwd, or pinned via `BEADS_DIR` (`README.md:186-198`). There is no `user_id` axis — like [[Profile__ByteRover]], the unit of memory is the project, not the person. Within a project:

- **ID prefix** (`bd-`, configurable; `bd-main-...` in the dogfood db) namespaces issues; cross-rig creation can override the prefix (`types.go:69-70`).
- **`source_repo`** / **`source_system`** mark multi-repo and federation provenance (`types.go:54, 68`).
- **Wisps** are the per-machine, non-synced scope — ephemeral local state that never leaves the clone.
- **Federation peers** (below) are the cross-org scope.

## Operational story — local-first, git-optional, federation-capable

- **Install once, use everywhere** (`README.md:17-37`): `brew install beads` / `npm i -g @beads/bd`, then `bd init` per project. The repo is a *tool* you install, not a dependency you vendor (`README.md:33`).
- **Git-optional** (`README.md:179-208`): set `BEADS_DIR` + `bd init --stealth` (sets `no-git-ops: true`) and every core command runs with zero git calls — for Sapling/Jujutsu/Piper repos, monorepo subdirs, CI, or ephemeral `/tmp` databases.
- **Agent onboarding:** `bd init` writes/updates `AGENTS.md` so agents discover the workflow; `bd setup <agent>` installs richer integration (hooks, skills) for Claude Code, Codex, Factory.ai Droid, Cursor, mux, and more. An **MCP server** ships as `beads-mcp` on PyPI.
- **Federation** (`FEDERATION-SETUP.md`): peer-to-peer sync of whole databases over Dolt's distributed VC, with **data-sovereignty tiers** (T1/T2/… for GDPR / regional compliance) stored per peer in `federation_peers.sovereignty`. `bd federation add-peer <name> <endpoint>`.
- **Contributor vs maintainer** (`README.md:81-84`): `bd init --contributor` routes planning issues to a separate repo (e.g. `~/.beads-planning`) so a fork's experimental task graph stays out of PRs.

## How this compares to the rest of the study

| Axis | Beads | Volt | ByteRover | Graphiti |
|---|---|---|---|---|
| **What is the memory of?** | The agent's own work graph (tasks + deps) | Conversation log (verbatim turns) | Curated knowledge (markdown topics) | Conversational episodes |
| **Memory kind** | **Procedural / working** | Episodic (lossless) | Semantic (curated) | Episodic + semantic |
| **Storage primitive** | Dolt (versioned SQL graph) | Embedded Postgres DAG | Markdown tree + sidecar + git | Cypher graph DB (4 backends) |
| **Recall** | SQL views + graph traversal | Summary nodes + raw recent | BM25 + LLM synth | BM25 + cosine + BFS rerank |
| **Vector search?** | **None** | No | No | Yes (cosine) |
| **Write policy** | Mutate + immutable `events` audit | Verbatim append + async summary | LLM-curated, HITL-gated | LLM episode→graph extraction |
| **Supersession** | `supersedes` graph edge | n/a | explicit MERGE op | bi-temporal edge invalidation |
| **Eviction** | Haiku memory-decay + reversible snapshots | Dolt/Upward modes | "dream" prune/synth | edge `invalid_at` |
| **Multi-writer** | Yes — hash IDs + Dolt cell-merge | Process-local | Per-project + cloud sync | DB-dependent |
| **License** | MIT | GPL-3.0 (per its profile) | Elastic-2.0 | Apache-2.0 |
| **On-disk truth** | Dolt DB (JSONL is export only) | Postgres | Markdown files | Graph DB |

The cleanest pairing is **Beads vs Volt**: both are coding-agent memory, both lean on a versioned database, and the Dolt-name collision is a trap (Volt's *eviction mode* named "Dolt" vs. Beads *built on* Dolt the DB). But Volt remembers the **conversation**; Beads remembers the **plan**. The cleanest contrast on *kind of memory* is **Beads vs everything-vector** (Mem0 / MemPalace / Supermemory / Hindsight / RetainDB): Beads is the entry that proves an "agent memory" product need not contain a single embedding if the thing worth remembering is structured work.

## Working-checklist answers

Per the study's reading rubric (`README.md:17-35`):

- **Storage topology.** Single store: Dolt (Git-like versioned MySQL-compatible SQL), embedded in-process by default or external `dolt sql-server` in server mode. No vector/KV/graph-DB split — the graph is SQL tables (`issues` + `dependencies`).
- **Write policy.** Mutate the `issues` row, append to immutable `events`. Atomic claims. Dolt commit history + cell-level merge per write. Supersession via `supersedes` edge. No LLM in the CRUD path; LLM only for compaction.
- **Scopes & namespacing.** Project directory (`.beads/`, walk-up or `BEADS_DIR`); ID prefix; `source_repo`/`source_system`; per-machine wisps; federation peers with sovereignty tiers. No user_id axis.
- **Schema of a memory.** ~50-column `issues` row (`0001` / `types.go:16-105`): five structured content fields, status/priority/type enums, full timestamp set, compaction metadata, JSON `metadata`, coordination fields. Typed `dependencies` edges (~19 kinds). `events` audit trail. `config` KV for `bd remember`.
- **Recall surface.** SQL: `bd ready` (`ready_issues` view), `bd blocked`, `bd show`, `bd list`, `bd dep tree`, `bd prime`, `bd memories`. `--json` everywhere. Graph traversal + cycle detection, no semantic search.
- **Eviction & compaction.** `internal/compact`: Haiku-summarized "memory decay" of closed issues (Tier 1+), originals preserved in `issue_snapshots`/`compaction_snapshots`, refuses to grow size; wisp TTL by `wisp_type`.
- **Serialization on disk.** Dolt DB under `.beads/` is authoritative. `.beads/issues.jsonl` is a plain-JSON export for viewing/interchange — **not** the source of truth, not a full backup. Sync via `bd dolt push/pull` on `refs/dolt/data`.
- **Operational story.** Local-first, install-once CLI; git-optional (`BEADS_DIR` + `--stealth`); MCP server (`beads-mcp`); `bd setup` integrations; peer-to-peer federation with data-sovereignty tiers.

## Pointers worth following

- `internal/storage/schema/migrations/0001_create_issues.up.sql` — the full issue schema in one file; pair with `internal/types/types.go:16-105` for the typed view.
- `internal/types/types.go:776-812` — the dependency-edge taxonomy. Reading this is the fastest way to see that Beads is a typed knowledge graph wearing an issue-tracker's clothes.
- `internal/idgen/hash.go:55-85` — hash-ID generation; the conflict-free-multi-agent story in 30 lines.
- `internal/compact/compactor.go` + `internal/compact/haiku.go` — the memory-decay engine and the exact Haiku prompt (`haiku.go:264-291`).
- `cmd/bd/memory.go` + `cmd/bd/prime.go` — the literal `remember`/`recall`/`prime` surface; the "survive compaction via SessionStart hook re-injection" mechanism.
- `internal/storage/schema/migrations/0037_uuid_primary_keys.up.sql` — read the header comment for the federation-collision rationale that drives the whole ID design.
- `FEDERATION-SETUP.md` — the distributed / data-sovereignty story; the only entry in the study with a GDPR-tier knob.

## Open questions for the study

- **Is procedural memory in-scope?** Beads stretches the study's definition of "memory layer" — it's a task tracker, not a recall engine. The right framing is probably that the study is implicitly about *episodic/semantic* memory, and Beads is the data point that names the missing axis (procedural/working memory). Worth stating that boundary explicitly in the README.
- **No semantic recall — is that a gap or a thesis?** Beads bets that the durable, structured part of agent memory needs SQL, not embeddings. Where does that bet break? (Probably: "find the issue about X" when the agent doesn't remember the ID — there's no vector fallback.)
- **The `interactions` table is unmined.** It logs every LLM call (model/prompt/response/tool/exit). That's an agent-telemetry store sitting inside an issue tracker — structurally the same as Lossless's `claude-code-tool-traces`. Is anything *reading* it for recall, or is it write-only audit?
- **Federation vs. the other entries' multi-tenancy.** Beads' peer-to-peer Dolt sync with sovereignty tiers is a different multi-writer model from Honcho's peer/observer scoping or Mem0's user/agent/run axes. The comparison is worth a dedicated note.
- **Provenance of the project.** GitHub org `gastownhall`, but the Go module path is `github.com/steveyegge/beads` and badges point at `steveyegge/beads`; LICENSE attributes "Beads Contributors." Confirm the canonical home/maintainer before citing externally.
