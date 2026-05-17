---
name: Neo Profile
slug: neo
upstream: https://github.com/Parslee-ai/neo
package: neo-reasoner (PyPI)
license: Apache-2.0
maintainer: Parslee AI
study: studies/memory-layers-for-agents
profile_path: studies/memory-layers-for-agents/neo
profile_kind: cli + library + claude-code-plugin + codex-plugin
date_created: 2026-05-17
---

# Neo — Profile

A profile of Neo as it lives in this study (`studies/memory-layers-for-agents/neo/`). Cites pinned paths so you can jump to source rather than trust paraphrase. Read this alongside [`Profile__Mem0.md`](./Profile__Mem0.md) for the general-memory contrast and [`Profile__Volt.md`](./Profile__Volt.md) for the immutable-store contrast.

## TL;DR

Neo is a **self-improving code-reasoning engine with persistent semantic memory**, packaged as a CLI (`neo "<query>"`), a Python library (`from neo import NeoEngine`), a Claude Code plugin (`/neo` and five specialized agents: `neo-review`, `neo-optimize`, `neo-architect`, `neo-debug`, `neo-pattern`), and a parallel Codex plugin sharing the same `~/.neo/` storage.

The interesting bit is not the embeddings. It is that Neo treats memory as a **deterministic supersession chain over typed, scoped, append-only facts**, *and* closes the learning loop by detecting whether its suggestions actually helped the codebase (architectural quality metrics — import cycles, god files, nesting depth — not user thumbs-up). Confidence isn't "did the user accept it," it's "did the codebase get healthier." That feedback signal feeds back into recall ranking, so familiar queries cost less reasoning effort over time.

Storage is **plain scoped JSON files** at `~/.neo/facts/` (`facts_global.json`, `facts_org_{id}.json`, `facts_project_{id}.json`) with embeddings stored inline (`src/neo/memory/store.py:70`, `src/neo/memory/models.py:112, 136-138`). No vector DB. Atomic tempfile-rename writes. This is the most "audit a memory record by reading the file" architecture in the study — closest in spirit to our `context-vigilance` discipline, but with an actual recall API on top.

## Why this exists — the design bet

Neo's bet is that for a reasoning agent, **memory is the product, not a feature**. Most memory layers (Mem0, LangMem) are designed to slot under any agent. Neo inverts that: the engine and the memory are designed together, the same way a database engine and its storage layer are designed together. That gives Neo permission to make choices a general layer can't:

1. **Typed facts.** Seven kinds: `CONSTRAINT`, `ARCHITECTURE`, `PATTERN`, `REVIEW`, `DECISION`, `KNOWN_UNKNOWN`, `FAILURE` (`src/neo/memory/models.py:94-113`). Recall layers always include constraints unconditionally; other types are ranked.
2. **Deterministic supersession.** Cosine > 0.85 between a new fact and an existing one triggers supersession with a `superseded_by` ↔ `supersedes` pointer pair, +0.05 confidence carryover (capped at 1.0), and a `needs_review=True` cascade onto facts that `depends_on` the superseded one (`store.py:925-933`).
3. **Outcome-driven confidence.** Outcomes are new `REVIEW` facts that link back to the originals via `suggestion_fact_ids`; success_count and confidence boost on the originals when the codebase improves (`outcomes.py:57`, `CHANGELOG.md:10`).
4. **Reasoning-effort gating.** High-confidence familiar queries run with `reasoning.effort="low"` on gpt-5 models (cheaper); novel queries run `"high"` / `"xhigh"` (more thinking). Memory directly monetizes into inference cost (`CHANGELOG.md:7`).

That last one is the killer insight: **memory pays for itself by reducing the size of the inference bill on familiar problems**, not by increasing accuracy on novel ones. Most memory layers can't claim this.

## The fact record schema

```python
@dataclass
class Fact:
    id: str                          # UUID[:16]
    subject: str                     # concise label
    body: str                        # full content
    kind: FactKind                   # CONSTRAINT|ARCHITECTURE|PATTERN|REVIEW|DECISION|KNOWN_UNKNOWN|FAILURE
    scope: FactScope                 # GLOBAL|ORG|PROJECT|SESSION
    org_id: str
    project_id: str
    is_valid: bool                   # True until superseded
    superseded_by: Optional[str]     # supersession chain forward
    supersedes: Optional[str]        # supersession chain backward
    depends_on: list[str]            # references to other fact IDs
    needs_review: bool               # set when a dependency is superseded
    embedding: Optional[np.ndarray]  # 768-dim Jina Code v2
    tags: list[str]                  # "seed"|"community"|"synthesized"|...
    metadata: FactMetadata           # see below
```
(`src/neo/memory/models.py:94-113`)

`FactMetadata` (`models.py:56-77`) carries `created_at`, `last_accessed`, `access_count`, `source_file`, `source_prompt` (first 200 chars), `confidence ∈ [0,1]`, `success_count`, and `provenance ∈ {"structural", "inferred", "observed"}`.

Compare to Mem0 (`Profile__Mem0.md` §schema): Neo's record carries supersession pointers, dependency edges, confidence, success counts, provenance, and a fact kind — all of which Mem0 omits. Neo is heavier per record; that weight is what supports its retrieval discipline.

## Scopes — auto-detected from git

Four levels (`src/neo/memory/models.py:47-52`):

| Scope | Cap | Source |
|---|---|---|
| `GLOBAL` | ~200 facts | Cross-project idioms |
| `ORG` | ~100 facts | Git remote regex (`scope.py:33-99`) — GitHub/GitLab/Azure DevOps formats; "unknown" fallback |
| `PROJECT` | ~500 facts | SHA-256[:16] of codebase root path (`scope.py:102-115`) |
| `SESSION` | ~50 facts | Per-invocation; dies with the process |

Scope caps are hard. When a scope exceeds its cap, `_enforce_scope_limit()` (`store.py:590-634`) evicts the lowest-scoring facts. The tags `{"seed", "community", "synthesized"}` are protected from eviction (`store.py:61, 619`) — this is how curated knowledge survives churn from new arrivals.

The org auto-detection from git remote is a small but excellent design choice: team conventions live at org scope without anyone explicitly tagging them.

## Write policy — append + supersede + cascade

`add_fact(...)` embeds the new fact, then runs a cosine-similarity search against the same scope (`src/neo/memory/store.py:35`, `models.py:35`):

- If max similarity > **0.85**, supersede: mark the existing fact `is_valid=False`, set `superseded_by` / `supersedes`, carry confidence forward `+0.05`.
- Cascade: any fact with `depends_on` pointing at the superseded one gets `needs_review=True` (`store.py:935`).
- Otherwise: append the new fact.

`process_outcomes(...)` runs after each session by comparing what Neo suggested to what actually shipped (git diff). Accepted suggestions boost `success_count` and confidence on the originating facts. Modified suggestions trigger pattern extraction (a new fact, tagged `outcome`). Outcomes are capped at 5 per session, 50 per project (`outcomes.py:30`, `store.py:58`) to keep the loop from drowning in noise during active development.

The supersession-with-cascade pattern is the closest thing the study has to "the database knows its facts have an order." Mem0 doesn't model this. Volt's immutable log avoids needing it (every assertion is preserved in time order). StateBench will *test* you on it — see [`Profile__StateBench.md`](./Profile__StateBench.md) §failure modes.

## Recall API

```python
def retrieve_relevant(
    self,
    query: str,
    query_embedding: Optional[np.ndarray] = None,
    k: int = 5,
    max_tokens: int = 12000,
) -> ContextResult
```

Returns a `ContextResult` with facts organized into layers: **constraints** (always included), **valid_facts** (ranked), **invalidated** (separated, with supersession pointers), **known_unknowns**, **environment** (`context.py:55-76`).

Ranking is:

```
score = cos_sim(query, fact.embedding)
      * confidence
      * success_bonus(success_count)   # log₂, capped at +0.2 (models.py:21-33)
      + recency_boost                  # facts touched this session
```

Scope priority: global constraints first, then project, then org, then session.

Also exposes:

- **`construct search/list/show`** — semantic search over the bundled **Construct** pattern library (`CONSTRUCT.md:72, 159-177`). FAISS-indexed, sub-100 ms.
- **`detect_implicit_feedback`** — between-session comparison of consecutive prompts to infer satisfaction (`engine.py:150-158`). Re-asked → confidence down; novel-after-related → confidence up.

The "invalidated" layer is unusual and important. Most memory APIs hide superseded facts entirely. Neo surfaces them so the agent can say "this used to be true, and the reason it changed was X" — exactly the kind of provenance StateBench rewards.

## Eviction & compaction — multiple coordinated mechanisms

Unlike Mem0 (none) or Volt (deterministic compaction loop), Neo has **four** distinct mechanisms:

1. **Scope-limit eviction** (`store.py:590-634`). Hard caps; evict lowest-scoring (excluding protected tags).
2. **Stale pruning** (`prune_stale_facts()`). Age ≥ 14 days **and** confidence < 0.4 (`store.py:47-54`).
3. **Demotion** (`demote_unhelpful_facts()`). `access_count ≥ 10` with no successes → confidence -= 0.1 (floor 0.1).
4. **Synthesis** (`synthesize_reviews()`). Group REVIEW facts with cosine > 0.80 into archetypal patterns (`store.py:36`). Removes redundant noise without losing signal.

Plus `purge_dead_facts()` to reclaim disk space from invalid facts with no live dependents.

This is the most thoughtful eviction story in the study. It's also where Neo's "memory engine" framing pays off — a memory *layer* can't justify this much policy; a memory *engine* has to.

## Serialization on disk

The format is the contract. JSON v2.0, one fact per object, embedding floats inline:

```json
{
  "version": "2.0",
  "facts": [
    {
      "id": "abc123f4567890ab",
      "subject": "Docker layer caching order",
      "body": "In Dockerfiles, copy dependency manifests before source so that...",
      "kind": "pattern",
      "scope": "global",
      "is_valid": true,
      "confidence": 0.85,
      "embedding": [0.123, -0.456, ...],
      "metadata": {
        "created_at": 1715284800.0,
        "success_count": 3,
        "provenance": "community"
      }
    }
  ]
}
```

(`src/neo/memory/store.py:866-877`, `src/neo/memory/models.py:119-138`)

Atomic tempfile-rename writes prevent partial writes on crash. Global facts are re-read from disk before save (best-effort merge; last writer wins on same ID). Commit history of the JSON is the audit trail (`README.md:634`).

This is the file format Neo's design wants you to be able to `cat`, `grep`, `jq`, and commit to git — which makes it the most `context-vigilance`-adjacent member of the study even though its mechanics are more sophisticated.

## Operational story — local-first, plugin-distributed

| Entry point | Use |
|---|---|
| `neo "query"` | One-shot CLI invocation |
| `from neo import NeoEngine` | Python library |
| Claude Code `/neo` + 5 sub-agents | Plugin (`.claude-plugin/plugin.json`) |
| Codex plugin (parity) | Same six skills, Codex CLI wrapper |

Local files at `~/.neo/`:

- `facts/` — three JSON files (global/org/project)
- `sessions/` — session records (drives outcome detection on next run)
- `config.json` — provider, model, memory backend
- `construct_index.faiss` — pattern library index (built on first `construct search`)

Privacy posture (`README.md:168-169`): codebase embeddings stay on-device via `fastembed`; only prompts + retrieved facts leave the machine. Same threat model as using an LLM directly — no codebase-text-to-vendor leak.

Providers: OpenAI, Anthropic, Google, Ollama, local — selected via `~/.neo/config.json`. Default model selection includes gpt-5.5 and `claude-sonnet-4-5-20250929` (the project is well-maintained against current models).

## MapCoder — the multi-agent inner loop

Neo implements a Solver / Critic / Verifier loop adapted from MapCoder (Islam et al. 2024):

1. **Difficulty estimation** — input size + keywords → easy/medium/hard.
2. **Time budget** — 30 / 60 / 120 seconds.
3. **Plan phase** — seed minimal `PlanStep`s with `preconditions/actions/exit_criteria/failure_signatures/verifier_checks` (`models.py:50-62`); expand only if blocked.
4. **Simulation phase** — `SimulationTrace` (`models.py:65-71`) runs traces, collects failure signatures.
5. **Code generation** — executable blocks (not diffs).
6. **Verification** — static checks + constraint validation against the recalled constraint layer.
7. **Early exit** — if confidence > 0.8 *and* simulation consensus *and* clean static checks (`engine.py:197-200`).

The interesting bit for this study isn't the MapCoder mechanics themselves; it's that the memory layer is consulted *inside* this loop (constraints in step 6, patterns in step 5, environment all through). Memory shapes reasoning; reasoning generates outcomes; outcomes shape memory. That's the closed loop.

## The Construct

`/construct/` is a curated, vendor-agnostic **pattern library** (`CONSTRUCT.md`). Patterns are markdown files (<300 lines, mandatory author, structured sections: Intent / Forces / Solution / Consequences / References). FAISS-indexed for sub-100 ms search (`construct/` + `src/neo/construct.py`).

This is *separate* from the fact store. Facts are what Neo learns; Constructs are what Neo was taught. The split is the right call — agent-written facts and human-curated patterns have different quality bars, different review processes, different lifetimes.

## What's inside this submodule

| Path | What's there |
|---|---|
| `src/neo/cli.py` | CLI entry point (~1000 LOC) |
| `src/neo/engine.py` | `NeoEngine` orchestrating MapCoder |
| `src/neo/memory/store.py` | The fact store (~1478 LOC, the real heart) |
| `src/neo/memory/{models,scope,outcomes,context,constraints}.py` | Schema, scope detection, outcome detection, context assembly, constraint logic |
| `src/neo/construct.py` + `/construct/` | Pattern library + index |
| `src/neo/adapters/` | OpenAI / Anthropic / Google / Ollama / local LM adapters |
| `community_facts.json` | 10-fact pre-curated seed library (Docker caching, GraphQL N+1, K8s probes, CORS, Postgres VACUUM, …) |
| `tests/` | 40+ pytest files |
| `docs/` | Architecture guides |
| `.claude-plugin/` | Claude Code plugin manifest + agent files |
| `.agents/plugins/marketplace.json` + `plugins/neo/` | Codex plugin |
| `specs/` | Project-level design specs (not configuration — design docs) |
| `CHANGELOG.md`, `INSTALL.md`, `QUICKSTART.md`, `QUESTIONS.md` | Unusually thorough for a project this size |

Read order if you want the core: `models.py` → `store.py` → `engine.py` → `CHANGELOG.md` (the design history is recorded there).

## Mental model for using it well

- **Curate constraints early.** Constraint facts are always included in every recall. A weak constraint layer means a weak agent.
- **Treat the supersession chain as documentation.** Don't `purge_dead_facts` aggressively in active codebases — the "this used to be true" trail is what stops the agent from re-suggesting reverted patterns.
- **Use outcome detection.** It's the part most users skip. Without it, Neo is just a smart vector lookup; with it, the confidence numbers become meaningful.
- **Don't dump everything in `global` scope.** Project scope has a 500-fact cap and is where most lessons belong. Global is for cross-project idioms.
- **Protect curated facts with tags.** `seed` / `community` / `synthesized` are eviction-immune. New community-contributed patterns should get the tag.
- **Reach for the Construct pattern library before writing a fact.** If something is worth teaching, it might be worth teaching everyone — that's what Construct is for.

## When NOT to reach for this

- **General agent memory across many users.** Neo's scope axis is `org/project`, not `user`. For "remember things about Alice," reach for Mem0.
- **You don't want a CLI in the loop.** Neo is designed to be invoked. If you want a memory layer that any process can call via HTTP, this isn't it.
- **You don't trust the cosine-0.85 threshold for your domain.** Supersession is deterministic in code but the threshold is a single global constant. Domains with semantically-close-but-substantively-different facts will mis-supersede.
- **Markdown-and-git is enough.** For pure project knowledge meant for humans, `context-vigilance` produces nicer artifacts. Neo's value lights up when the agent's choice of *which* fact to surface matters.

## How this compares to our own `context-vigilance` skill

Neo and `context-vigilance` are closer than any other pair in the study:

| | Neo | context-vigilance |
|---|---|---|
| **Storage** | JSON files at `~/.neo/facts/` | Markdown files at `<repo>/context-v/` |
| **Audit** | Commit JSON in git (works) | Commit markdown in git (works) |
| **Schema** | Typed facts with embedding + supersession pointers | Frontmatter with type, version, wikilinks |
| **Recall** | Semantic + confidence × success | Human + grep |
| **Supersession** | Cosine-0.85 deterministic | Human edits + `superseded_by` wikilink (by convention) |
| **Scope** | Auto-detected from git remote | Directory tree (`prompts/`, `specs/`, …) |
| **Best fit** | Agent recall ranking, with humans curating constraints | Human + agent reading, with structure as the value |

You'd use them together: `context-vigilance` for the durable project knowledge humans review in PRs; Neo for the ranked, scoped, agent-surfaced facts the engine accumulates as it works. They are *not* alternatives — they sit at different points on the human-readability ↔ machine-rankability axis.

## One-line summary

> Neo is a typed, scoped, supersession-aware fact store with an outcome-driven confidence loop that monetizes memory into inference-cost savings — packaged with a MapCoder reasoning engine and distributed as Claude Code and Codex plugins, with the most file-readable storage in the study.
