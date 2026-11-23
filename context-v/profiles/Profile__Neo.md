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
site_uuid: 11b30fa0-5fca-4346-ad43-7b5e576dadbb
hex_code: 2aytzx
date_authored_initial_draft: 2026-05-17
date_authored_current_draft: 2026-11-23
lede: >-
  Familiar high-confidence queries get dialled down to low reasoning effort —
  memory shows up as a smaller bill, not better accuracy.
summary: >-
  Profile of Neo (Parslee AI) as pinned in the memory-layers-for-agents study.
  Neo was retired upstream on 2026-11-08 (final release 0.63.0). It is the
  study's most file-readable storage and its most thoughtful eviction
  story, and it sits closest to the context-vigilance discipline of any entry
  besides git-mode Letta. Covers the typed fact kinds (seven, now eight), the four
  git-auto-detected scopes with hard caps, deterministic supersession at cosine
  0.85 with a needs-review cascade onto dependents, outcome-driven confidence
  from architectural quality metrics, the layered ContextResult that deliberately
  surfaces invalidated facts rather than hiding them, the coordinated eviction
  mechanisms (synthesis since removed), the on-disk JSON v2.0 format, the MapCoder inner loop, and the
  Construct pattern library kept separate from learned facts. Read the
  invalidated-layer and supersession sections when the requirement is
  state-correctness over time.
publish: true
title: Neo — Profile
date_modified: 2026-11-23
authors:
  - Michael Staton
status: Reference
type: Profiles
---

# Neo — Profile

> **Retired upstream.** Parslee AI retired Neo on 2026-11-08 with a final release, `neo-reasoner` 0.63.0, and says development continues in CAR (`README.md:4`, `CHANGELOG.md:3-7`). The README says the repository is archived; existing installs keep working but get no fixes. This profile now describes a finished design, which is still useful as a reference for typed, supersession-aware fact memory.

A profile of Neo as it lives in this study (`studies/memory-layers-for-agents/neo/`). Cites pinned paths so you can jump to source rather than trust paraphrase. Read this alongside [`Profile__Mem0.md`](./Profile__Mem0.md) for the general-memory contrast and [`Profile__Volt.md`](./Profile__Volt.md) for the immutable-store contrast.

## TL;DR

Neo is a **self-improving code-reasoning engine with persistent semantic memory**, packaged as a CLI (`neo "<query>"`), a Python library (`from neo import NeoEngine`), a Claude Code plugin (`/neo` and five specialized agents: `neo-review`, `neo-optimize`, `neo-architect`, `neo-debug`, `neo-pattern`), and a parallel Codex plugin sharing the same `~/.neo/` storage.

The interesting bit is not the embeddings. It is that Neo treats memory as a **deterministic supersession chain over typed, scoped, append-only facts**, *and* closes the learning loop by detecting whether its suggestions actually helped the codebase (architectural quality metrics — import cycles, god files, nesting depth — not user thumbs-up). Confidence isn't "did the user accept it," it's "did the codebase get healthier." That feedback signal feeds back into recall ranking, so familiar queries cost less reasoning effort over time.

Storage is **plain scoped JSON files** at `~/.neo/facts/` (`facts_global.json`, `facts_org_{id}.json`, `facts_project_{id}.json`) with embeddings stored inline (`src/neo/memory/store.py:493-495`, `src/neo/memory/models.py:442, 512-526`). No vector DB. Atomic tempfile-rename writes. This is the most "audit a memory record by reading the file" architecture in the study — closest in spirit to our `context-vigilance` discipline, but with an actual recall API on top.

## Why this exists — the design bet

Neo's bet is that for a reasoning agent, **memory is the product, not a feature**. Most memory layers (Mem0, LangMem) are designed to slot under any agent. Neo inverts that: the engine and the memory are designed together, the same way a database engine and its storage layer are designed together. That gives Neo permission to make choices a general layer can't:

1. **Typed facts.** Seven kinds at the original pin: `CONSTRAINT`, `ARCHITECTURE`, `PATTERN`, `REVIEW`, `DECISION`, `KNOWN_UNKNOWN`, `FAILURE`; 0.18.0 added an eighth, `EPISODE`, for single instance-specific events that later consolidate into patterns (`src/neo/memory/models.py:235-256`). Recall layers always include constraints unconditionally; other types are ranked.
2. **Deterministic supersession.** Cosine > 0.85 between a new fact and an existing one triggers supersession with a `superseded_by` ↔ `supersedes` pointer pair, +0.05 confidence carryover (capped at 1.0), and a `needs_review=True` cascade onto facts that `depends_on` the superseded one (`src/neo/memory/store.py:4041-4060`, `:4108`).
3. **Outcome-driven confidence.** Outcomes are new `REVIEW` facts that link back to the originals via `suggestion_fact_ids`; success_count and confidence boost on the originals when the codebase improves (`src/neo/memory/outcomes.py:247`, `CHANGELOG.md:1427-1436`, 0.16.0 entry). Later releases made promotion stricter: a lesson becomes durable only after two git-verified acceptances on two distinct revisions (`CHANGELOG.md`, 0.54.1 and 0.62.5 entries).
4. **Reasoning-effort gating.** High-confidence familiar queries run with `reasoning.effort="low"` on gpt-5 models (cheaper); novel queries run `"high"` / `"xhigh"` (more thinking). Memory directly monetizes into inference cost (`CHANGELOG.md:1427-1436`, 0.16.0 entry; the mapping is `effort_from_memory` at `src/neo/reasoning_effort.py:47-71`: three or more patterns at average confidence ≥ 0.8 → `low`).

That last one is the killer insight: **memory pays for itself by reducing the size of the inference bill on familiar problems**, not by increasing accuracy on novel ones. Most memory layers can't claim this.

## The fact record schema

```python
@dataclass
class Fact:
    id: str                          # UUID[:16]
    subject: str                     # concise label
    body: str                        # full content
    kind: FactKind                   # CONSTRAINT|ARCHITECTURE|PATTERN|REVIEW|DECISION|KNOWN_UNKNOWN|FAILURE|EPISODE
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
(`src/neo/memory/models.py:423-555`; the record has since gained domain, episode-context and bi-temporal fields)

`FactMetadata` (`src/neo/memory/models.py:288-360`) carries `created_at`, `last_accessed`, `access_count`, `source_file`, `source_prompt` (first 200 chars), `confidence ∈ [0,1]`, `success_count`, and `provenance ∈ {"structural", "inferred", "observed"}`.

Compare to Mem0 (`Profile__Mem0.md` §schema): Neo's record carries supersession pointers, dependency edges, confidence, success counts, provenance, and a fact kind — all of which Mem0 omits. Neo is heavier per record; that weight is what supports its retrieval discipline.

## Scopes — auto-detected from git

Four levels (`src/neo/memory/models.py:259-264`, caps at `src/neo/memory/store.py:79-84`):

| Scope | Cap | Source |
|---|---|---|
| `GLOBAL` | ~200 facts | Cross-project idioms |
| `ORG` | ~100 facts | Git remote regex (`src/neo/memory/scope.py:51-143`) — GitHub/GitLab/Azure DevOps formats; "unknown" fallback |
| `PROJECT` | ~500 facts | SHA-256[:16] of the normalized `git remote get-url origin`, falling back to the codebase root path (`src/neo/memory/scope.py:184-217`); old path-keyed files are migrated |
| `SESSION` | ~50 facts | Per-invocation; dies with the process |

Scope caps are hard. When a scope exceeds its cap, `_enforce_scope_limit()` (`src/neo/memory/store.py:3101`) evicts the lowest-scoring facts. The tags `{"seed", "community", "synthesized"}` are protected from eviction (`src/neo/memory/store.py:144`) — this is how curated knowledge survives churn from new arrivals.

The org auto-detection from git remote is a small but excellent design choice: team conventions live at org scope without anyone explicitly tagging them.

## Write policy — append + supersede + cascade

`add_fact(...)` embeds the new fact, then runs a cosine-similarity search against the same scope (`SUPERSESSION_THRESHOLD` at `src/neo/memory/store.py:59`, `add_fact` at `:638`):

- If max similarity > **0.85**, supersede: mark the existing fact `is_valid=False`, set `superseded_by` / `supersedes`, carry confidence forward `+0.05`.
- Cascade: any fact with `depends_on` pointing at the superseded one gets `needs_review=True` (`src/neo/memory/store.py:4108`).
- Otherwise: append the new fact.

`process_outcomes(...)` runs after each session by comparing what Neo suggested to what actually shipped (git diff). Accepted suggestions boost `success_count` and confidence on the originating facts. Modified suggestions trigger pattern extraction (a new fact, tagged `outcome`). Outcomes are capped at 5 per session, 50 per project (`src/neo/memory/outcomes.py:37`, `src/neo/memory/store.py:118`) to keep the loop from drowning in noise during active development.

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

That was the signature at the original pin. At the final pin `FactStore.retrieve_relevant(query, k=30, domain=None)` returns a ranked fact list (hybrid dense + BM25, then half by rank score and half by raw cosine, `src/neo/memory/store.py:764-790`), and context assembly wraps it. Context assembly returns a `ContextResult` with facts organized into layers: **constraints** (always included), **valid_facts** (ranked), **invalidated** (separated, with supersession pointers), **known_unknowns**, **environment** (`src/neo/memory/context.py:55-146`, `src/neo/memory/models.py:557-567`).

Ranking is:

```
score = cos_sim(query, fact.embedding)
      * confidence
      * success_bonus(success_count)   # log₂, capped at +0.2 (src/neo/memory/models.py:28-35)
      + recency_boost                  # facts touched this session
```

Scope priority: global constraints first, then project, then org, then session.

Also exposes:

- **`construct search/list/show`** — semantic search over the bundled **Construct** pattern library (`CONSTRUCT.md:72, 159-177`). FAISS-indexed, sub-100 ms.
- **`detect_implicit_feedback`** — between-session comparison of consecutive prompts to infer satisfaction (now `src/neo/memory/store.py:2634`). Re-asked → confidence down; novel-after-related → confidence up.

The "invalidated" layer is unusual and important. Most memory APIs hide superseded facts entirely. Neo surfaces them so the agent can say "this used to be true, and the reason it changed was X" — exactly the kind of provenance StateBench rewards.

## Eviction & compaction — multiple coordinated mechanisms

Unlike Mem0 (none) or Volt (deterministic compaction loop), Neo had **four** distinct mechanisms at the original pin. The fourth, synthesis, was deleted in 0.39.0 (commit `1e926bd`): upstream reported it ran four months on a live install, minted no PATTERN facts, and kept re-consuming its own output.

1. **Scope-limit eviction** (`src/neo/memory/store.py:3101`). Hard caps; evict lowest-scoring (excluding protected tags).
2. **Stale pruning** (`prune_stale_facts()`). Age ≥ 14 days **and** confidence < 0.4 (`src/neo/memory/store.py:87-88`, `:3621`).
3. **Demotion** (`demote_unhelpful_facts()`). At the final pin it has two tiers: `access_count` 5-9 with no successes → confidence -= 0.1 (floor 0.1); `access_count ≥ 10` with no successes → marked invalid (`src/neo/memory/store.py:3687-3700`).
4. **Synthesis** (`synthesize_reviews()`, removed in 0.39.0). Grouped REVIEW facts by cosine into archetypal patterns. Its replacement is a pair of hard caps: newest 50 `independent` facts and newest 25 attempt-outcome facts per project (`src/neo/memory/store.py:118-128`, `:3777`).

Plus `purge_dead_facts()` (`src/neo/memory/store.py:3336`) to reclaim disk space from invalid facts with no live dependents, and `retire_feed_facts()` (`:3450`), which invalidates the facts minted by the seed and community feeds removed in 0.61.0.

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

(`src/neo/memory/store.py:3900-3920`, `src/neo/memory/models.py:489-555`)

Atomic tempfile-rename writes prevent partial writes on crash. Global facts are re-read from disk before save (best-effort merge; last writer wins on same ID). If you keep `~/.neo/facts/` under version control, commit history of the JSON is the audit trail; `neo memory audit` (`README.md:1028`) reports on the store without that.

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

Privacy posture (`README.md:143`): codebase embeddings stay on-device via `fastembed`; only prompts + retrieved facts leave the machine. Same threat model as using an LLM directly — no codebase-text-to-vendor leak.

Providers: OpenAI, Anthropic, Google, Ollama, local — selected via `~/.neo/config.json`. Default OpenAI model is `gpt-5.6` at the final pin (`src/neo/config.py:195`); the Anthropic adapter still defaults to `claude-sonnet-4-5-20250929` (`src/neo/adapters.py:569`). A CAR adapter routes through Parslee's CAR runtime.

## MapCoder — the multi-agent inner loop

Neo implements a Solver / Critic / Verifier loop adapted from MapCoder (Islam et al. 2024):

1. **Difficulty estimation** — input size + keywords → easy/medium/hard.
2. **Time budget** — 30 / 60 / 120 seconds.
3. **Plan phase** — seed minimal `PlanStep`s with `preconditions/actions/exit_criteria/failure_signatures/verifier_checks` (`src/neo/models.py:240`); expand only if blocked.
4. **Simulation phase** — `SimulationTrace` (`src/neo/models.py:274`) runs traces, collects failure signatures.
5. **Code generation** — executable blocks (not diffs).
6. **Verification** — static checks + constraint validation against the recalled constraint layer.
7. **Early exit** — if confidence > 0.8 *and* simulation consensus *and* clean static checks (`EARLY_EXIT_CONFIDENCE` at `src/neo/engine.py:180`, checked at `:1435`).

The interesting bit for this study isn't the MapCoder mechanics themselves; it's that the memory layer is consulted *inside* this loop (constraints in step 6, patterns in step 5, environment all through). Memory shapes reasoning; reasoning generates outcomes; outcomes shape memory. That's the closed loop.

## The Construct

`/construct/` is a curated, vendor-agnostic **pattern library** (`CONSTRUCT.md`). Patterns are markdown files (<300 lines, mandatory author, structured sections: Intent / Forces / Solution / Consequences / References). FAISS-indexed for sub-100 ms search (`construct/` + `src/neo/construct.py`).

This is *separate* from the fact store. Facts are what Neo learns; Constructs are what Neo was taught. The split is the right call — agent-written facts and human-curated patterns have different quality bars, different review processes, different lifetimes.

## What's inside this submodule

| Path | What's there |
|---|---|
| `src/neo/cli.py` | CLI entry point (~1,870 LOC) |
| `src/neo/engine.py` | `NeoEngine` orchestrating MapCoder |
| `src/neo/memory/store.py` | The fact store (~4,300 LOC at the final pin, up from ~1,478; the real heart) |
| `src/neo/memory/{models,scope,outcomes,context,constraints}.py` | Schema, scope detection, outcome detection, context assembly, constraint logic |
| `src/neo/memory/{observer,transcript,episodes}.py` | Background observer that mines Claude Code, Codex and merged-GitHub-PR transcripts into probationary lessons (added after the original pin) |
| `src/neo/construct.py` + `/construct/` | Pattern library + index |
| `src/neo/adapters.py` | OpenAI / Anthropic / Google / Ollama / local / CAR LM adapters |
| ~~`community_facts.json`~~ | Removed in 0.61.0 with the seed feed and `neo contribute` (commit `c68b772`); upstream measured community facts in 0 of 141 prompts |
| `tests/` | ~145 test files |
| `docs/` | Architecture guides |
| `.claude-plugin/` | Claude Code plugin manifest + agent files |
| `.agents/plugins/marketplace.json` + `plugins/neo/` | Codex plugin |
| `plugins/cursor-neo/` | Cursor plugin (added after the original pin) |
| `specs/` | Project-level design specs (not configuration — design docs) |
| `CHANGELOG.md`, `INSTALL.md`, `QUICKSTART.md`, `QUESTIONS.md` | Unusually thorough for a project this size |

Read order if you want the core: `models.py` → `store.py` → `engine.py` → `CHANGELOG.md` (the design history is recorded there).

## Mental model for using it well

- **Curate constraints early.** Constraint facts are always included in every recall. A weak constraint layer means a weak agent.
- **Treat the supersession chain as documentation.** Don't `purge_dead_facts` aggressively in active codebases — the "this used to be true" trail is what stops the agent from re-suggesting reverted patterns.
- **Use outcome detection.** It's the part most users skip. Without it, Neo is just a smart vector lookup; with it, the confidence numbers become meaningful.
- **Don't dump everything in `global` scope.** Project scope has a 500-fact cap and is where most lessons belong. Global is for cross-project idioms.
- **Protect curated facts with tags.** `seed` / `community` / `synthesized` are eviction-immune (`src/neo/memory/store.py:144`). The seed and community feeds themselves were removed in 0.61.0, so the tags now matter mainly for facts you add by hand.
- **Reach for the Construct pattern library before writing a fact.** If something is worth teaching, it might be worth teaching everyone — that's what Construct is for.

## When NOT to reach for this

- **General agent memory across many users.** Neo's scope axis is `org/project`, not `user`. For "remember things about Alice," reach for Mem0.
- **You don't want a CLI in the loop.** Neo is designed to be invoked. If you want a memory layer that any process can call via HTTP, this isn't it.
- **You don't trust the cosine-0.85 threshold for your domain.** Supersession is deterministic in code but the threshold is a single global constant. Domains with semantically-close-but-substantively-different facts will mis-supersede.
- **Markdown-and-git is enough.** For pure project knowledge meant for humans, `context-vigilance` produces nicer artifacts. Neo's value lights up when the agent's choice of *which* fact to surface matters.
- **You need a maintained dependency.** Neo was retired on 2026-11-08; 0.63.0 is the final release and gets no fixes. Read it for the design; adopt CAR or another entry for running code.

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

## Changes since the previous pin (2026-05-02 → 2026-11-08)

Pin moved from `3d1bb22cc8ce` to `af51f759041f`: 363 commits, releases 0.17.0 through 0.63.0. License is still Apache-2.0.

- **Retired.** The last commit retires Neo: 0.63.0 is the final release, every reasoning run prints a retirement notice, the PyPI classifier moves to Inactive, and the README says the repository is archived and development continues in CAR (`README.md:4`, `CHANGELOG.md:3-7`, `src/neo/cli.py`).
- **The core memory model held.** Typed facts, four git-derived scopes with the same caps (`src/neo/memory/store.py:79-84`), supersession at cosine 0.85 with +0.05 carry-forward and the `needs_review` cascade (`:59`, `:4041-4108`), the layered `ContextResult` with an invalidated layer (`src/neo/memory/models.py:557-567`), JSON `"version": "2.0"` on disk (`src/neo/memory/store.py:3911`), and memory-sized reasoning effort (`src/neo/reasoning_effort.py:47-71`).
- **New fact kind.** `EPISODE` (0.18.0) stores single events with when/where/why/who context; agreeing episodes consolidate into patterns (`src/neo/memory/models.py:235-256`).
- **Learning moved to a background observer.** An observer daemon mines Claude Code and Codex transcripts and merged GitHub PR reviews into probationary lessons (`src/neo/memory/observer.py`, `src/neo/memory/transcript.py`). Promotion to durable memory needs two git-verified acceptances on two distinct revisions; from 0.62.5 a fix Claude writes its own way at the flagged lines counts if static checks pass.
- **Eviction changed.** REVIEW→PATTERN synthesis was deleted in 0.39.0 (commit `1e926bd`); demotion gained a second tier that invalidates facts read 10+ times with no success (`src/neo/memory/store.py:3687-3700`); the seed and community feeds and `neo contribute` were removed in 0.61.0 (commit `c68b772`), and `community_facts.json` is gone.
- **Recall changed.** Hybrid dense + BM25 retrieval over the whole valid corpus with query decomposition (`src/neo/memory/store.py:764-790`), short follow-ups matched against the previous reply (0.61.0), and lessons sent once per session.
- **Project identity.** `PROJECT` scope now hashes the normalized git remote, so clones and worktrees share one fact file; path-keyed files are migrated (`src/neo/memory/scope.py:184-217`).
- **Surfaces.** A Cursor plugin (`plugins/cursor-neo/`), a Claude Code Mods workspace pane (0.56.0) and in-conversation lines (0.58.0), and CAR Lattice membership (0.54.0).

## One-line summary

> Neo is a typed, scoped, supersession-aware fact store with an outcome-driven confidence loop that monetizes memory into inference-cost savings — packaged with a MapCoder reasoning engine and distributed as Claude Code and Codex plugins, with the most file-readable storage in the study.
