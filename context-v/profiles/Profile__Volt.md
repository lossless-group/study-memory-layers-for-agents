---
name: Volt Profile
slug: volt
upstream: https://github.com/Martian-Engineering/volt
package: voltcode (CLI)
license: MIT
maintainer: Martian Engineering (Voltropy)
study: studies/memory-layers-for-agents
profile_path: studies/memory-layers-for-agents/volt
profile_kind: coding-agent + embedded-postgres + dual-state-context
date_created: 2026-05-17
site_uuid: 5a8b5cff-a6ee-4751-8329-c913b3d76002
hex_code: seo1hr
date_authored_initial_draft: 2026-05-17
date_authored_current_draft: 2026-05-17
lede: >-
  Numeric thresholds decide when to compact, and the model is only ever asked
  to write summaries. Long-horizon coding as a DB problem.
summary: >-
  Profile of Volt (Martian Engineering) as pinned in the
  memory-layers-for-agents study. It is the only entry that is a full coding
  agent with memory built in rather than a memory layer with agents bolted on,
  and the most database-shaped design in the study. Covers the dual-state
  Lossless Context Management architecture (verbatim immutable Postgres log plus
  a materialized DAG of summary nodes), the sprig/bindle/dN hierarchy and its
  lineage pointers, soft and hard token thresholds with three-level escalation,
  the Dolt-versus-Upward runtime modes, content-addressed large-file handling
  with precomputed exploration summaries, and the LLM-Map and Agentic-Map
  engine-level iteration primitives. The transferable lesson called out in the
  file is the threshold-plus-escalation pattern: parameterize the control loop
  with explicit numbers instead of letting the model decide.
publish: true
---

# Volt — Profile

A profile of Volt as it lives in this study (`studies/memory-layers-for-agents/volt/`). Cites pinned paths so you can jump to source rather than trust paraphrase. Read alongside [`Profile__Mem0.md`](./Profile__Mem0.md), [`Profile__Neo.md`](./Profile__Neo.md), [`Profile__StateBench.md`](./Profile__StateBench.md). Volt is the only entry in the study that is a **full coding agent** with memory built in (rather than a memory layer with optional agents on top).

## TL;DR

Volt is an OpenCode fork from Martian Engineering / Voltropy that introduces **Lossless Context Management (LCM)** — a deterministic, database-backed dual-state architecture for keeping a coding agent productive across indefinitely-long sessions.

The two-state design (`README.md:20-32`) is the whole bet:

1. **Immutable store.** Every user message, assistant response, and tool result is persisted **verbatim** in embedded PostgreSQL and **never modified**. Ground truth.
2. **Active context.** A high-fanout **DAG of summary nodes** (sprigs → bindles → dN) computed over the immutable history. A *materialized cache*, not a source of truth.

The summarization control loop is **deterministic** — soft and hard token thresholds drive it, not the model. A **three-level escalation** (normal → aggressive → deterministic fallback with no LLM) guarantees convergence (`README.md:31-32`). Two runtime modes (`README.md:237-251`): **Dolt** (evicts oldest bindles, "ghost cue" lineage pointers let you retrieve them on-demand) and **Upward** (recursive bottom-up condensation at unbounded depth, no eviction, default).

Operator-level recursion (`LLM-Map`, `Agentic-Map`) pushes iteration and concurrency from the stochastic model down into the deterministic engine (`README.md:44-50`). Large files are stored externally with content-addressed IDs + precomputed exploration summaries — never loaded into active context (`README.md:53-54`).

Claimed wins: higher OOLONG long-context scores than Claude Code at every length between 32K and 1M tokens, using Opus 4.6 (`README.md:57-59`); infinite sessions; zero compaction-wait latency.

If you read one sentence: **Volt argues that long-horizon coding is a database problem, not a prompt-engineering problem, and ships the DAG, the Postgres schema, and the deterministic control loop to prove it.**

## Why this exists — the design bet

Everyone else in this space lets the model decide what to remember. Volt's bet (`README.md:20-21`):

> Rather than asking the model to invent a memory strategy, LCM provides a deterministic, database-backed infrastructure.

That sentence is the whole thesis. The model is asked to write *summaries* (a focused task it's good at); the engine decides *when* to summarize, *what* to summarize, *which level* to summarize at, *what to evict* (Dolt) or *not evict* (Upward), and *how to reassemble* the active context on every turn. This division of labor is the opposite of the "agentic memory" pattern where the model loops on memory ops.

Four design moves follow:

1. **Verbatim persistence.** No information is ever lost, even when active context is compressed. This makes the system *retrievable* in a way summary-only systems aren't.
2. **DAG over flat history.** Newer messages stay raw; older ones get rolled up into summary nodes that can themselves be rolled up. Unbounded depth (Upward) or capped with eviction-plus-pointers (Dolt).
3. **Deterministic thresholds + escalation.** Numbers in config, not vibes. `0.6` soft threshold, `1.2×` hard threshold, three-level escalation including an LLM-free fallback (`config.ts:172-202`).
4. **Engine-level iteration.** `LLM-Map` and `Agentic-Map` make the engine the place loops live, so the model never burns context tracking a `for i in items` it could have delegated.

## The dual-state design in the code

Postgres tables (`db.ts:113-145`):

| Table | Holds |
|---|---|
| `messages` | role, content, token_count, created_at, seq |
| `message_parts` | 16+ part types (text, reasoning, tool, patch, file, subtask, compaction, step_start/finish, snapshot, agent, retry) |
| `summaries` | content, token_count, kind, summary_level, condensation_order, summary_type, is_off_context |
| `summary_messages` | maps summaries → source messages |
| `summary_parents` | DAG parent-child edges |
| `summary_lineage_pointers` | archive_stub / archive_full / lineage_parent |
| `context_items` | position, item_type (message | summary) |
| `large_files` | file_id (content-addressed SHA-256[:16]), storage_kind (path / inline_text / inline_binary), token_count, exploration_summary |

Notice what's there and what isn't: every message persisted forever; summaries with explicit level + condensation_order; lineage pointers so an evicted bindle still has an address; large files separated entirely from the message stream.

## The DAG — sprigs, bindles, dN

(`db.ts:83-84, 199-206, 288-315`)

- **Sprig** — L1 summary (condensation_order=1) over raw messages / leaves.
- **Bindle** — L2 summary (condensation_order=2) over sprigs.
- **dN** — higher-order condensations (d3, d4, …), unbounded in Upward.
- **archive_stub** — short off-context pointer node for an evicted bindle (Dolt only).

Sprig/bindle are the UX labels (Dolt-flavored); internally everything is `d{N}` with `condensation_order`. Each summary node has a deterministic SHA-256-based `summary_id`, tracks `token_count`, `file_ids` it touches, `is_off_context`, `created_at`. The lineage pointers (`db.ts:199-206`) preserve the retrieval path end-to-end: ghost-cue pointer → archive_stub → archive_full → original messages.

This is the most database-shaped memory design in the study. Where Neo's contract is "the JSON file" and Mem0's contract is "the vector store payload," Volt's contract is **the schema + the foreign-key graph between summary tables**. Wrapping the same retrieval logic on a different store would be a substantial rewrite.

## Soft / hard thresholds and the escalation protocol

Defaults (`config.ts:172, 177`):

- `DEFAULT_CTX_CUTOFF_THRESHOLD = 0.6` — 60% of the model's context window. Above this, compaction runs **asynchronously between turns** (`README.md:31`).
- `DEFAULT_CRITICAL_THRESHOLD_MULTIPLIER = 1.2` — 1.2× the soft threshold. If compaction fails to reduce token count, escalate.

Three levels (`compaction-escalation.ts`):

1. **Normal** summarization (`SUMMARY_MAX_OUTPUT_TOKENS = 2200` default).
2. **Aggressive** summarization (~60% of normal max output, tighter compression).
3. **Deterministic fallback** — no LLM. Literal truncation / bindle consolidation. Guaranteed to converge.

Per-lane thresholds in Dolt (`config.ts:182-202`):

```
Dolt leaves:  soft=50K, delta=5K, target=50K, cap=50K
Dolt sprigs:  soft=10K, delta=2K, target=10K
Dolt bindles: soft=10K, delta=2K, target=10K
Upward context threshold: 0.75
Upward leaf chunk: 20K, min 8 messages per sprig
```

The numbers themselves matter less than the fact that they exist as numbers — every knob is in config (or env, see env section below), every behavior is reproducible.

## Dolt vs Upward modes

(`README.md:237-251`, `strategy-dolt.ts:12-20`, `strategy.ts`)

| Behavior | Dolt | Upward |
|---|---|---|
| Compaction | Evict oldest bindles | Recursive d1→d2→d3→dN |
| Off-context retrieval | Available (via ghost cue + lineage pointers) | Disabled |
| `lcm_grep` (raw-message search) | Works | Works |
| Manual `/compact` | Creates one new bindle | Full recursive pass |
| Bindle eviction | Max 1 per cycle | Never |
| Env var | `VOLTCODE_LCM_MODE=dolt` | `VOLTCODE_LCM_MODE=upward` (default) |

**Ghost cue** (Dolt-only): a compact pre-response memory hint pointing at an off-context bindle, carrying `summary_id`, lineage pointers, and metadata. Lets the agent locate and expand archived content without exhaustive search. `lcm_expand_query` is the candidate-resolution-with-scoring API (`retrieval.ts`).

**Upward** is the default because it's simpler and avoids ghost-cue bookkeeping. Dolt's headline trick is the lineage-pointer-as-address pattern — an evicted bindle still has a name and can be brought back. Upward never needs that because it never evicts.

In study terms: Upward optimizes for "no information ever leaves the working set" (the StateBench dream); Dolt optimizes for "working set stays bounded, retrieval is the escape hatch."

## Large file handling

(`README.md:53-54`, `db.ts:146-159`, `large-file.ts:91-94`, `large-file-threshold.ts`)

Above a configurable token threshold, files are never loaded into active context. Instead Volt inserts a **compact reference**: content-addressed ID (`file_<sha256_first_16>`), original path, and a precomputed **Exploration Summary** generated by a MIME-type-aware dispatcher (Python / TypeScript / Go / Rust / …, under `explore/`).

This is the single most underrated design move in the codebase. Most coding agents poison their context the first time you ask about a 20K-line file. Volt structurally cannot.

## Operator-level recursion — LLM-Map / Agentic-Map

(`README.md:44-50`, `llm-map.ts:24-38`, `agentic-map.ts:27-45`)

**LLM-Map**: process each item in a JSONL file via independent LLM API call (pure function, no tools). Parameters: `input_path`, `output_path`, `prompt`, `output_schema` (JSON Schema), `model`, `concurrency` (default 16), `timeout_seconds`, `max_attempts`. Engine handles pool, retries, schema validation. Use case: classification, entity extraction, scoring.

**Agentic-Map**: spawn a full sub-agent session per item, concurrency 16. Each sub-agent has tool access (file read, web fetch, bash); `read_only` controls write permissions. Requires explicit task permission with `subagent_type: "agentic_map"` (`agentic-map.ts:89-98`).

Shared infrastructure (`map-shared.ts`): parses input JSONL, validates against schema (Zod-backed Draft 2020-12), registers output JSONL in the immutable store.

This pattern matters for the study even beyond Volt: it's the answer to "how do you do `for i in N items` without burning N×(growing context) tokens." You shouldn't. The engine should.

## What's inside this submodule

(Bun + TypeScript monorepo, Turbo-orchestrated)

| Path | What's there |
|---|---|
| `packages/voltcode/` | Core CLI / server (Bun + Hono backend) |
| `packages/voltcode/src/lcm/` | The LCM core: `db.ts`, `context.ts`, `condense.ts`, `summarize.ts`, `large-file.ts`, `explore/`, `integrity.ts`, `config.ts` |
| `packages/app/` | Shared web UI components (SolidJS + Tailwind) |
| `packages/desktop/` | Native desktop wrapper (Tauri v2) |
| `packages/plugin/` | Plugin API (`@opencode-ai/plugin`) |
| `packages/sdk/js/` | Generated TypeScript SDK (auto-generated from `server.ts`; regen via `./script/generate.ts`) |
| `packages/ui/`, `packages/util/` | Shared component library + utilities |
| `themes/` | `deltarune.json`, `undertale.json` (UI themes, semantic role mapping) |
| `sdks/vscode/` | VSCode extension |
| `infra/`, `sst.config.ts` | SST AWS deployment (Cloudflare home, Stripe + PlanetScale providers) |
| `install/` | Install scripts |
| `nix/`, `flake.nix` | Reproducible dev environments |
| `CLAUDE.md`, `STYLE_GUIDE.md`, `CONTRIBUTING.md`, `STATS.md` | Project conventions + download stats |

If you read three files: `README.md` (the LCM pitch, end to end), `packages/voltcode/src/lcm/db.ts` (the schema is the contract), `packages/voltcode/src/lcm/context.ts` (the assembly logic).

## Providers and models

(`provider.ts:61-84`)

Bundled: Anthropic, OpenAI (with custom `responses()` API for GPT-5+), Azure (responses or chat), Google, Google Vertex (with Anthropic fallback), OpenRouter, XAI, Mistral, Groq, DeepInfra, Cerebras, Cohere, Gateway, TogetherAI, Perplexity, Vercel, GitLab, Amazon Bedrock (credential chain), GitHub Copilot.

Config merge precedence (`README.md:150-157`):

1. Remote well-known config (auth)
2. Global config in `${XDG_CONFIG_HOME:-~/.config}/voltcode/`
3. `VOLTCODE_CONFIG` file path override
4. Project `voltcode.jsonc` / `voltcode.json` discovered upward
5. `VOLTCODE_CONFIG_CONTENT` inline JSON

Minimal config:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "enabled_providers": ["openai"],
  "model": "openai/gpt-5",
  "small_model": "openai/gpt-5-mini"
}
```

## UI modes (independent of LCM mode)

(`README.md:120-138`)

- **Agent UI mode** (TUI `Tab` toggle): `build` (read-write, full access) vs `plan` (read-only, asks permission before bash).
- **LCM runtime mode** (env): `dolt` vs `upward`. Controls memory compression, not UI.

These are deliberately independent axes. A `plan`-mode agent in `upward` LCM is a perfectly reasonable "explore a giant codebase without touching it" setup.

## Operational story

Embedded Postgres by default (`config.ts:4-54`):

- Default URL: `postgres://voltcode@127.0.0.1:54329/voltcode_lcm`
- Postgres 17.7, binaries at `${XDG_DATA_HOME:-~/.local/share}/voltcode/postgres/17.7/bin/`
- Data at `.../data/`, log at `${XDG_LOG_HOME:-~/.local/log}/voltcode/postgres.log`
- Lock at `.../install.lock`

External Postgres via `LCM_DATABASE_URL` (highest precedence). AWS RDS via `RDS_ENDPOINT`, `RDS_USERNAME`, `RDS_PASSWORD`, `RDS_PORT` (default 5432), `RDS_DATABASE` (default `voltcode_lcm`). Connection pool max 10, statement timeout 30s, max lifetime 30 minutes, prepared statements disabled for RDS Proxy compatibility (`db.ts:475-499`). Per-user schema isolation supported (`user-context.ts`).

Install:

```bash
curl -fsSL https://raw.githubusercontent.com/Martian-Engineering/volt/dev/install | bash
```

Adoption (`STATS.md`): 2.7M+ GitHub releases + npm packages by early 2026, with a January-2026 spike to 54K + 11K downloads in a single day.

## Performance & benchmark claim

`README.md:57-59`:

> Volt with LCM achieves higher scores than Claude Code on the OOLONG long-context benchmark, including at every context length between 32K and 1M tokens, using Opus 4.6.

Independent verification would route through StateBench (see [`Profile__StateBench.md`](./Profile__StateBench.md)) — Volt's verbatim immutable log is structurally very strong against the hallucination failure mode, and Upward mode (no eviction) is structurally very strong against the resurrection failure mode. The architectural prediction is "Volt should score well on SFRR." Nobody has run it through StateBench yet that I can see.

## Mental model for using it well

- **Treat the immutable store as the contract.** Don't write code that assumes the active context is complete; it's a *cache*. If you need a fact for sure, the message_parts table has it.
- **Default to Upward.** Dolt is for cases where you genuinely want a bounded working set with off-context retrieval as the escape hatch. Most users want Upward.
- **Use `LLM-Map` / `Agentic-Map` for any "do X for each item" task.** Resist the urge to let the model loop. The engine does it better.
- **Let large files stay large.** Don't `cat` them into the agent; let the exploration summary do its job, and let the agent pull specific regions when needed.
- **Tune thresholds in config, not by stripping context.** Every behavior is parameterized. `VOLTCODE_LCM_UPWARD_CONTEXT_THRESHOLD=0.85` is a legitimate choice; "manually editing the conversation" is not.
- **Watch the compaction-state events in the TUI.** When something feels slow, the live task tree (`README.md:308-312`) tells you whether you're waiting on a tool, an LLM call, or a background compaction.

## When NOT to reach for this

- **You want a memory layer for a non-coding agent.** Volt *is* the agent. Lift the LCM ideas, not the package.
- **You can't run embedded Postgres.** External Postgres is supported but adds operational weight. If your environment is "stateless container, no disk," LCM is the wrong fit.
- **Short sessions.** All the machinery costs zero overhead in theory, but you don't get value from it under ~10K tokens of history.
- **You need fine-grained per-user persona memory across many users.** Volt's scope axis is the conversation/session, not the user. Stack Mem0 on top for cross-session user memory if you need both.

## How this compares to the rest of the study

| Axis | Volt | Mem0 | Neo |
|---|---|---|---|
| **Storage** | Postgres DAG (immutable log + summary nodes + large_files) | Vector + entity store + SQLite | Scoped JSON files |
| **Write policy** | Verbatim append + async LLM summary at thresholds | Single-pass LLM extraction + entity link | Append + deterministic supersession |
| **Eviction** | Dolt: evict bindles + ghost cue. Upward: never | None in OSS | Multiple coordinated mechanisms |
| **Off-context retrieval** | First-class (Dolt lineage pointers) | N/A (vector recall is the only path) | Surfaces invalidated facts in `ContextResult` |
| **Iteration model** | Engine-level (LLM-Map / Agentic-Map) | Caller's loop | Caller's loop |
| **Best fit** | Long-horizon coding | Cross-session user memory | Code-reasoning with outcome learning |

The deepest design difference: Volt structurally prevents loss; Mem0 structurally accepts loss; Neo structurally tracks supersession. Three different bets about what "memory" even means for an agent.

## How this compares to our own `context-vigilance` skill

Volt is at the opposite end of the human-readability ↔ machine-management axis from `context-vigilance`.

`context-vigilance` is: a few dozen markdown files at `<repo>/context-v/`, hand-versioned, hand-edited, hand-reviewed in PRs. The human is the indexer; the agent reads files and greps. No DAG, no thresholds, no summarization, no Postgres. Everything in plain text, everything in git.

Volt is: embedded Postgres, a multi-level summary DAG, async background compaction with deterministic escalation, content-addressed large-file storage, engine-managed iteration. None of it human-curated; all of it agent-facing.

They aren't competitors — they answer different questions. `context-vigilance` answers "how should a team and its agents keep durable, reviewed knowledge about a project?" Volt answers "how does a coding agent stay coherent across a session that lasts a week?" You can run a Volt session inside a project that uses `context-vigilance` and the markdown files become tool-reads inside Volt's immutable log; the two compose cleanly.

The interesting transferable lesson from Volt back to `context-vigilance`: the soft/hard threshold pattern + deterministic escalation is a generally useful idea. If we ever build agent-facing summary indexes over `context-v/` directories, doing it with explicit numeric thresholds rather than "the LLM decides" is the lesson worth lifting.

## One-line summary

> Volt is the only entry in the study that treats long-horizon agent memory as a database problem and ships the schema to back it up — a deterministic, embedded-Postgres-backed DAG of summary nodes over a verbatim immutable message log, with two runtime compaction modes, engine-level iteration primitives, and content-addressed large-file handling that together make "infinite sessions" a real claim rather than a marketing one.
