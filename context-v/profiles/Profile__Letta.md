---
name: Letta Profile
slug: letta
upstream: https://github.com/letta-ai/letta
package: letta (PyPI)
license: Apache-2.0
maintainer: letta-ai (formerly MemGPT-ai; Charles Packer, Sarah Wooders et al.)
study: studies/memory-layers-for-agents
profile_path: studies/memory-layers-for-agents/letta
profile_kind: platform (server + agents) + library + hosted-cloud
date_created: 2026-05-17
site_uuid: 679199ab-f28a-474a-b7cb-bb45ffa4b54e
hex_code: 53u8t2
date_authored_initial_draft: 2026-05-17
date_authored_current_draft: 2026-05-17
lede: >-
  The one entry where the agent itself — not an extraction pipeline, not a
  background summarizer — is the memory manager.
summary: >-
  Profile of Letta (letta-ai, formerly MemGPT) as pinned in the
  memory-layers-for-agents study. It is the study's most lineage-heavy entry and
  its closest structural relative to the context-vigilance discipline. Covers
  the two-tier memory model (agent-editable core blocks rendered into the system
  prompt, pgvector-backed archival passages), the core_memory_append and
  core_memory_replace tools that are the MemGPT signature, block sharing via the
  blocks_agents junction table, the FastAPI + Postgres + pgvector platform with
  its OpenAI-compatible endpoint and built-in OpenTelemetry, the agent-profile
  system prompts, and the git-backed memory mode. The comparison table against
  context-vigilance is the section to read if an agent is ever going to write
  into a context-v directory directly.
publish: true
---

# Letta — Profile

A profile of Letta as it lives in this study (`studies/memory-layers-for-agents/letta/`). Cites pinned paths so you can jump to source rather than trust paraphrase. Read alongside [`Profile__Mem0.md`](./Profile__Mem0.md) and [`Profile__Neo.md`](./Profile__Neo.md) — Letta sits between them on the human-curated ↔ agent-self-curated axis, and is the most lineage-heavy entry in the study (it is the direct successor to MemGPT).

## TL;DR

Letta is "the platform for building stateful agents" (`README.md:1`) and the **direct successor to MemGPT** — the 2023 Berkeley paper that named the agent-memory problem and shipped the OS-inspired hierarchical-memory pattern (main context = RAM, archival = disk, recall = paging). The README is explicit: *"Letta (formerly MemGPT)."*

Mechanically, Letta is a **FastAPI server + Postgres + pgvector + agent framework** (`compose.yaml`, `letta/server/server.py`) rather than a library you import. Every agent is a persistent ORM row (`letta/orm/agent.py:45-46`); every memory **Block** is a Postgres record with versioning + history (`letta/orm/block.py:56-61`); every **Passage** in the archival tier carries an embedding column (`letta/schemas/passage.py:35-77`). Agents edit their own core memory via the famous MemGPT tools `core_memory_append` and `core_memory_replace` (`letta/schemas/memory.py:804-837`). Memory blocks are **shareable across agents** (junction table `blocks_agents`), which makes multi-agent shared state a database join rather than a sync protocol.

The platform exposes a REST API, a WebSocket streaming API, and an **OpenAI-compatible `/v1/chat/completions` endpoint** (`letta/server/rest_api/chat_completions_interface.py`) — so a Letta agent looks like a ChatGPT-shaped model to any client. Provider-agnostic on the LLM side (OpenAI / Anthropic / Google / Azure / Ollama / vLLM / Groq / Together / Mistral / DeepSeek / xAI). Built-in OpenTelemetry (`letta/otel/`) and a docker-compose stack make it operable as a real service from day one.

The newer **git-backed memory** mode (`letta/schemas/memory.py:76`, `letta/schemas/memory_repo.py:19-37`) is the most interesting recent move: agents whose `git_enabled=True` render their blocks as files in a real git repo (`system/persona.md`, `system/human.md`, `skills/web_search/SKILL.md`) and every memory edit becomes a commit. This is *unusually close to the `context-vigilance` discipline* — see comparison below.

If you want one sentence: **Letta is the operational descendant of MemGPT — a Postgres-backed agent platform where each agent is a persistent row, core memory is a set of agent-editable blocks rendered into the system prompt, archival memory is a pgvector-backed passage store with semantic search, and the whole thing ships as a FastAPI server with an OpenAI-compatible endpoint and a git-backed memory mode that makes memory edits look like commits.**

## Why this exists — the MemGPT thesis, productized

Three load-bearing claims from the MemGPT paper survive into Letta:

1. **Context windows are bounded; intelligence about what's in them shouldn't be.** Letta splits memory into a **core** tier (always in context, agent-editable) and an **archival** tier (paginated, on-demand, retrievable by search). The agent is taught, via system prompt + function definitions, that it can promote facts from archival into core and edit core in place.
2. **The agent should be the memory manager.** Not a background summarizer, not an extraction pipeline. The LLM holds the tools and decides when to write. `core_memory_append` and `core_memory_replace` are the headline tools (`letta/schemas/memory.py:804-837`).
3. **State must persist across sessions.** An agent isn't a chat object that lives in a process — it's a database record with `id`, `message_ids`, `llm_config`, `system`, and a many-to-many edge to blocks (`letta/orm/agent.py:45-90+`). Restart the server; the agent is exactly where you left it.

What Letta adds on top of the MemGPT paper:

- A real **platform** (server, REST, WebSocket, OpenAI-compatible API, OTel) instead of a research notebook.
- **Block sharing** between agents — a single Block row is joinable to N agents, so "Alice's preferences" can be one record that three agents reference.
- **Git-backed memory** for agents that want a real audit trail and a filesystem projection.
- **Multiple agent profiles** (`memgpt_v2_agent`, `sleeptime_agent`, `react_agent`, `letta_v1_agent`, `voice_chat`) each with its own system-prompt template.
- A hosted product (Letta Cloud) and a desktop client (Letta Code).

## The two-tier memory model

### Core memory — Blocks

(`letta/schemas/block.py:20-36`)

```python
class Block(BaseModel):
    value: str            # the actual text
    label: str            # e.g. "human", "persona", or a custom key
    limit: int            # char ceiling (default CORE_MEMORY_BLOCK_CHAR_LIMIT)
    read_only: bool       # whether the agent can edit it
    description: str      # what this block is for
```

A `Memory` object wraps a list of blocks and renders them into the system prompt as XML (`letta/schemas/memory.py:149-173`):

```xml
<memory_blocks>
  <human>
    <description>What we know about the user</description>
    <metadata>
      - chars_current=412
      - chars_limit=2000
    </metadata>
    <value>Name: Alice. Works at Acme. Prefers concise answers.</value>
  </human>
  <persona>...</persona>
</memory_blocks>
```

Three rendering modes (`letta/schemas/memory.py:688-732`): **standard** (XML), **line-numbered** (Anthropic-friendly, lets Claude reference exact lines for edits), and **git** (filesystem projection for `git_enabled` agents).

### Archival memory — Passages

(`letta/schemas/passage.py:35-77`)

```python
class Passage(BaseModel):
    text: str
    embedding: list[float]
    archive_id: Optional[str]
    tags: list[str]
    created_at: datetime
```

Stored in Postgres with pgvector. `PassageManager` (`letta/services/passage_manager.py`) handles CRUD and semantic search (OpenAI-embeddings by default, optional Redis cache). Pluggable backends include pgvector and TurboBuffer.

The agent interacts with archival via `archival_memory_insert` and `archival_memory_search` tools (semantic search returns top-k passages). This is the "paging" half of the MemGPT analogy.

### Block sharing across agents

The `blocks_agents` junction table (`letta/orm/agent.py`) means **a single Block can belong to many agents**. The implication: "Alice's preferences" lives in one row; Alice's research agent, planning agent, and code agent all reference it. Edits propagate. No sync logic to write.

This is the most underrated piece of Letta's design — every other entry in the study (Mem0, MemPalace, Neo) scopes memory to a user/agent/session axis and treats cross-agent sharing as out of scope. Letta makes it a database join.

## The self-editing tools — the MemGPT signature

```python
def core_memory_append(agent_state, label: str, content: str) -> None: ...
def core_memory_replace(agent_state, label: str, old_content: str, new_content: str) -> None: ...
```

(`letta/schemas/memory.py:804-837`, exposed via `letta/functions/function_sets/base.py`)

The agent invokes these like any other tool. The execution path is direct (no sandbox needed — they're modifying the agent's own state). The block's `value` is mutated, the limit checked, and on the next turn the new content renders into the system prompt.

This is the design move every other memory layer either avoided (Mem0, MemPalace store, never edit) or implemented differently (Neo's deterministic supersession at cosine > 0.85; Graphiti's temporal-window inactivation). Letta's bet is that the agent is the *only* entity with enough context to know what should change.

Failure mode worth flagging: the agent can forget to edit, edit wrongly, or thrash. This is exactly the territory StateBench (`Profile__StateBench.md`) measures.

## The platform layer

### Compose stack (`compose.yaml`)

```
letta_db      — postgres + pgvector v0.5.1
letta_server  — FastAPI (REST :8083, WebSocket :8283)
letta_nginx   — reverse proxy
```

This is the only entry in the study that ships a multi-service compose stack out of the box. Volt is close (embedded Postgres), but Letta is structured for *being a service other things hit over the network*.

### APIs

- **REST** (`letta/server/rest_api/routers/`) — full CRUD on agents, blocks, passages, archives, messages, groups.
- **WebSocket** (`letta/server/ws_api/`) — streaming I/O for agent runs.
- **OpenAI-compatible** (`letta/server/rest_api/chat_completions_interface.py`) — drop-in replacement for `/v1/chat/completions`. A Letta agent appears as a model to anything that speaks OpenAI.

The OpenAI-compatible surface is the cleanest path for adoption: any tool that talks to OpenAI can talk to a Letta agent without changes.

### Observability

`letta/otel/` ships:
- Decorator-based tracing (`tracing.py`)
- Prometheus-style metrics (`metrics.py`)
- SQLAlchemy auto-instrumentation (`sqlalchemy_instrumentation.py`)
- OTLP exporter wired by env var (`LETTA_OTEL_EXPORTER_OTLP_ENDPOINT`)

This is the only entry in the study where you can answer "what just happened" with a tracing waterfall in Grafana/Datadog without writing instrumentation yourself.

## Persistence model — every agent is a row

(`letta/orm/agent.py:45-90+`)

```
agents
  id (PK)                       — globally unique
  agent_type                    — memgpt_v2_agent | sleeptime_agent | react_agent | letta_v1_agent | voice_chat
  message_ids (json)            — IDs of in-context messages
  llm_config                    — endpoint, context window, sampling
  system                        — compiled system prompt

blocks
  id (PK)                       — versioned, history-tracked
  value, label, limit, read_only, description
  current_history_entry_id      — pointer for audit/rollback

blocks_agents (M:N)             — block-to-agent edges (sharing)

passages
  id, text, embedding (pgvector), archive_id, tags, created_at

archives                        — collections of passages
groups, groups_blocks           — multi-agent groups with shared blocks
messages                        — per-agent message history; group_id for multi-agent
tools, tools_agents             — tool definitions + assignments
```

Migrations: ~100+ Alembic files under `alembic/versions/`. Schema evolution is taken seriously.

## Git-backed memory — the recent, important move

(`letta/schemas/memory.py:76`, `letta/schemas/memory_repo.py:19-37`)

For agents with `git_enabled=True`, Letta stores memory as a **real git repository**. Blocks render as files:

```
system/persona.md
system/human.md
skills/web_search/SKILL.md
```

Every memory edit becomes a commit (`MemoryCommit` schema captures author + timestamp + file deltas). Version control, rollback, blame — for the agent's memory. This is a Letta Cloud feature in the current release but the schema lives in the OSS codebase.

This is genuinely novel for the study and worth thinking about: every other entry treats memory as either rows in a DB or chunks in a vector store. Letta-with-git treats memory as a filesystem with a history. That makes memory **reviewable in PRs** in a way nothing else here is. Closest comparison in our world: `context-vigilance` — markdown files committed to git, read by humans and agents. Letta-with-git is what happens if you let the *agent* be the committer.

## Agent profiles (the system-prompt templates)

(`letta/prompts/system_prompts/`)

| Profile | Use |
|---|---|
| `memgpt_v2_agent` | Classic MemGPT prompt — core memory + archival + recall tools |
| `sleeptime_agent` | Newer refinement; background-style memory management |
| `react_agent` | ReAct reasoning style |
| `letta_v1_agent` | Current canonical |
| `voice_chat` | Optimized for voice input |

Each is rendered via `Memory.compile()` (`letta/schemas/memory.py:688-732`) which assembles core memory + tool rules + archival summaries + function definitions into one string the LLM sees.

## What's inside this submodule

| Path | What's there |
|---|---|
| `letta/schemas/` | Pydantic schemas — `block.py`, `memory.py`, `passage.py`, `agent.py`, `archive.py`, `memory_repo.py` |
| `letta/orm/` | SQLAlchemy ORM models with versioning + history |
| `letta/server/` | FastAPI server (`server.py`), REST routers, WebSocket API, OpenAI-compatible endpoint |
| `letta/services/` | `passage_manager.py`, `tool_executor/`, `agent_serialization_manager.py`, group manager, ... |
| `letta/functions/function_sets/base.py` | Memory tools and other base tools the agent gets |
| `letta/prompts/system_prompts/` | Per-agent-type prompt templates |
| `letta/otel/` | Tracing, metrics, OTLP exporter |
| `alembic/` | ~100+ schema migrations |
| `compose.yaml`, `dev-compose.yaml`, `docker-compose-vllm.yaml` | Production / dev / vLLM stacks |
| `sandbox/` | Tool execution sandbox (subprocess / E2B / Modal) |
| `examples/` | Working examples |
| `tests/` | Pytest suite |
| `WEBHOOK_SETUP.md` | Webhook integration docs |

If you read three files: `letta/schemas/memory.py` (the whole memory model fits here), `letta/orm/agent.py` (how persistence is shaped), `letta/server/rest_api/chat_completions_interface.py` (the OpenAI-compatible adapter that's most likely your integration point).

## Provider support

(`letta/schemas/providers.py`)

OpenAI, Anthropic, Google AI + Vertex AI, Azure OpenAI, Ollama, vLLM, LM Studio, Groq, Together, Mistral, DeepSeek, xAI. All mapped to an `LLMConfig` (model, context window, endpoint, credentials).

The `line_numbered` block rendering (`letta/schemas/memory.py:698-702`) is enabled specifically for Anthropic + certain agent types — Claude does cleaner exact-line edits when blocks are numbered.

## Mental model for using it well

- **Pick the right agent profile up front.** `agent_type` determines the prompt template, the available memory tools, and the rendering mode. Hard to change later without confusing the agent.
- **Use blocks deliberately.** "Human" and "persona" are the canonical pair; resist creating a block per fact. Many small blocks fragment context; a few well-shaped blocks give the agent room to edit coherently.
- **Treat archival as the working set, not the long tail.** Archival is paginated and semantically searched — it's where "things the agent might need later" go. Don't dump the entire web there.
- **Share blocks across agents.** A single "user preferences" block joined to all of a user's agents beats every other propagation strategy.
- **Wire OTel.** This is the only entry in the study where you'll regret *not* turning on observability. Memory thrash, tool failures, prompt-compilation surprises — all visible only via tracing.
- **Consider git-backed memory if review matters.** It's heavier (you're running a real git repo per agent) but it's the path that produces auditable, blame-able memory state.
- **Use the OpenAI-compatible endpoint to test integration.** It's the lowest-resistance way to swap a Letta agent into something that already talks to OpenAI.

## When NOT to reach for this

- **You don't want to run a server.** Letta is operationally a service. If you want a library you import, look at Mem0 (Library mode) or MemPalace (MCP server is the surface but the engine is in-process).
- **Your "agents" are stateless RAG pipelines.** All of Letta's machinery is in service of *persistent* agents that survive restarts and learn over time. For one-shot retrieval, this is overkill.
- **You don't trust the agent to be its own memory manager.** Self-editing memory means the agent decides. Some failure modes (resurrection, hallucinated edits) are this design's tax. See StateBench's failure modes (`Profile__StateBench.md`) — particularly the *stale reasoning* and *resurrection* axes.
- **You need typed, queryable relationships.** Letta has blocks + passages + tags, not a graph. For "who knows whom" or "what depends on what," Graphiti is the structural fit.
- **You're on a closed model (Claude API, GPT API) and want zero infra.** Letta needs Postgres + the server. You can use it that way (the OSS stack runs locally) but if you want zero infra at all, Mem0's hosted platform is the comparable choice.

## How this compares to the rest of the study

| Axis | Letta | Mem0 | MemPalace | Neo | Volt | Graphiti |
|---|---|---|---|---|---|---|
| **Shape** | FastAPI platform + agents | Library / server / hosted | MCP + library + CLI | CLI + plugin | Coding agent | Library + REST + MCP |
| **Storage** | Postgres + pgvector | Vector + entity store + SQLite | Chroma + SQLite graph | JSON files | Postgres DAG | Graph DB (4 backends) |
| **Write policy** | **Agent self-edits via tools** | LLM extraction (single-pass) | Verbatim chunks | Deterministic supersession | Verbatim log + LLM summary | LLM extraction + dedupe |
| **Who decides what changes?** | **The agent** | The extraction prompt | Nobody (append only) | The threshold | The compaction loop | The dedup judge |
| **Cross-agent sharing** | **Block-level (DB join)** | Per-scope | Per-wing | Per-scope | Per-conversation | Per-`group_id` |
| **Persistence unit** | One agent = one row | Memory = many rows | Drawer = one chunk | Fact = one entry | Message = one row | Edge = one row |
| **Audit story** | History table + optional git | `history` SQLite | `filed_at` + git | git on JSON | Immutable message log | Bi-temporal columns |
| **Best fit** | Stateful agent platforms | Cross-session user memory | Long-tail verbatim recall | Code-reasoning loops | Long-horizon coding | Domains with real relationships |

The crucial axis Letta owns is **the agent is the memory manager** — that bet is the MemGPT inheritance and it cuts a different problem space than every other entry's "the framework decides" or "the user decides" or "the supersession algorithm decides."

## How this compares to our own `context-vigilance` skill

Letta-with-git-enabled is structurally the closest the study has to `context-vigilance`. Both store memory as **markdown files in a git repository**, both treat commits as the audit trail, both let humans review changes in PRs. The differences:

| | Letta (git-enabled) | context-vigilance |
|---|---|---|
| **Committer** | The agent (via `core_memory_append`/`replace`) | The human (via editor + git) |
| **File layout** | Determined by block labels (`system/persona.md`, `skills/.../SKILL.md`) | Determined by directory roles (`prompts/`, `specs/`, `blueprints/`, ...) |
| **Schema** | Pydantic blocks with `label` / `limit` / `read_only` / `description` | Markdown frontmatter (`semantic_version`, `tags`, `status`, ...) |
| **Recall** | Renders blocks into system prompt + archival semantic search | Human + agent grep + (optionally) Chroma over the markdown |
| **Versioning** | Git commits per memory edit | epoch.major.minor.patch in frontmatter + git commits |
| **Review** | The agent's edit history is grep-able | PRs are the review surface |

The transferable lesson: if we ever let an agent write to `context-v/` directly, **Letta's git-backed memory is the model to copy**. Block-as-file mapping, every edit as a commit, history table for rollback, descriptions explaining what each block is for. The `context-vigilance` discipline already produces the right artifacts; Letta shows what the agent-side commit pipeline looks like.

The honest reverse lesson — what `context-vigilance` could teach Letta: **frontmatter as the contract**. Letta's blocks have `description` fields, but no semantic-version, no status, no first-class wikilink graph between blocks. The `context-vigilance` frontmatter conventions would make Letta blocks dramatically more navigable.

## One-line summary

> Letta is the operational descendant of MemGPT — a Postgres + pgvector + FastAPI platform where every agent is a persistent row, core memory is a set of agent-editable blocks rendered into the system prompt, archival memory is a paginated semantic-search tier, and the agent itself is the memory manager via `core_memory_append`/`core_memory_replace`; plus an OpenAI-compatible API, built-in OpenTelemetry, multi-agent block sharing as a database join, and a git-backed-memory mode that turns memory edits into commits and brings the design as close to `context-vigilance` as the study gets.
