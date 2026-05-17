---
name: StateBench Profile
slug: statebench
upstream: https://github.com/Parslee-ai/statebench
package: statebench (Python)
license: MIT
maintainer: Parslee, LLC
study: studies/memory-layers-for-agents
profile_path: studies/memory-layers-for-agents/statebench
profile_kind: benchmark + harness + reference-implementations
date_created: 2026-05-17
---

# StateBench — Profile

A profile of StateBench as it lives in this study (`studies/memory-layers-for-agents/statebench/`). Cites pinned paths so you can jump to source rather than trust paraphrase. Unlike the other three entries, StateBench is **not a memory implementation** — it is the yardstick. Read it alongside [`Profile__Mem0.md`](./Profile__Mem0.md), [`Profile__Neo.md`](./Profile__Neo.md), and [`Profile__Volt.md`](./Profile__Volt.md) — those are the systems StateBench was built to evaluate.

## TL;DR

StateBench is a **conformance test for stateful AI agents** that operationalizes the abstract goal "state correctness over time" into six concrete failure modes and four metric families, then ships ten reference implementations spanning the obvious memory architectures so newcomers can see which design choices break which failure mode.

The dataset is 1,410 long-horizon **timelines** — sequences of conversation turns, state writes (with explicit fact IDs, sources, authority levels), supersessions, environment signals, and a query with ground truth — split 60/15/15/10 train/dev/test/hidden with 10 canary items for contamination detection (`data/releases/v1.0/metadata.json:1-18`, `docs/V1_SPEC.md:68-159`). Scoring is **hybrid** (`docs/EVALUATION.md:10-14`): deterministic phrase matching for unambiguous constraints, LLM-as-judge for paraphrase detection, multi-seed protocol for official submissions.

The reference implementation that motivates the whole project is **Memgine** — Parslee's deterministic memory engine — which on Opus 4.6 / v1.0 dev split hits **97.3% decision accuracy** with **37.1% superseded-fact resurrection** vs `state_based`'s 62.9% accuracy (`README.md:53-65`). The benchmark exists in large part to show that the state-based architecture from Liotta 2025/2026 (the two papers under `docs/`) outperforms transcript-replay, RAG, summary, fact-extraction, and fact-extraction-with-supersession baselines on the failure modes that actually matter for production agents.

**Why this matters for the study:** every other entry here is an *implementation* with a self-reported benchmark. StateBench is the independent harness that lets you not trust those self-reports.

## The six failure modes

These are the framing concepts the whole benchmark organizes around (`README.md:105-155`):

| # | Failure mode | What it means |
|---|---|---|
| 1 | **Resurrection** | System references facts that were explicitly invalidated |
| 2 | **Hallucination** | System asserts state that was never established |
| 3 | **Scope leak** | Information crosses scope/tenant/session boundaries |
| 4 | **Stale reasoning** | System ignores corrections in downstream reasoning |
| 5 | **Authority violation** | Lower-authority sources override higher (peer overrides policy) |
| 6 | **Temporal decay failure** | Time-sensitive state treated as permanent |

This taxonomy is the most useful artifact StateBench produces, even before you run a single benchmark. It gives you a vocabulary for what your agent gets wrong, and it maps cleanly to the architectural choices in Mem0 (resurrection-exposed), Neo (resurrection-resistant via supersession), and Volt (hallucination-resistant via immutable log).

A central finding (`docs/EVALUATION.md:96-116`): systems that include **more context** achieve higher decision accuracy but **also higher resurrection rates** — a fundamental trade. Memgine's whole claim is that supersession-tracking + relevance filtering breaks the trade.

## Metrics

Four families (`README.md:211-219`):

- **SFRR** — Superseded Fact Resurrection Rate. Target 0%.
- **Decision Accuracy** — correct yes/no/value.
- **Must Mention Rate** — required phrases appear.
- **Must Not Mention Violation Rate** — forbidden phrases don't appear.

Scoring details (`docs/EVALUATION.md:25-94`):

```
SFRR = (queries with must_not_mention violations) / (queries with any must_not_mention constraints)
     # deterministic only via contains_phrase()

Decision Accuracy:
  # First: deterministic extraction (yes/no signal words)
  # YES signals: "yes", "go ahead", "proceed", "approved", "can do", "will do"
  # NO signals:  "no", "don't", "do not", "cannot", "should not", "shouldn't", "stop", "hold off"
  # First-appearing signal wins for binary decisions
  # Fallback: LLM judge if inconclusive

Must Mention:
  # Deterministic contains_phrase (case-insensitive, regex, common paraphrase patterns)
  # LLM paraphrase fallback if deterministic fails

Must Not Mention:
  # Deterministic ONLY (strict, avoids LLM-false-negatives)
```

The asymmetry on Must Not Mention is deliberate and good: a benchmark that lets an LLM judge "no the model didn't *really* mention the forbidden thing" gives systems an easy way to game the score. Phrase-match-only on the forbidden side keeps the scoreboard honest.

## The dataset

Each timeline is a sequence of typed events (`docs/V1_SPEC.md:68-159, 113-206`):

- **conversation turns** (role + content)
- **state writes** with fact IDs, sources, **authority level** (policy / executive / manager / peer / subordinate / system / unverified), and **scope** (global / task / hypothetical / draft)
- **supersession events** — explicit (`Supersession` typed event) or implicit (inferred from natural language)
- **environment signals** (time, location, sensor)
- **queries** with ground truth (decision, must-mention, must-not-mention)

Difficulty tiers (`V1_SPEC.md:52`): easy / medium / hard / adversarial.

Track categories (`README.md:189-208`, `V1_SPEC.md:775-828`): supersession (explicit and implicit), durability (commitments survive interruptions), access control (scope/authority/privacy), temporal (freshness), integrity (identity/contradiction), and an adversarial track that combines failure modes.

The dataset format makes the failure modes **operational** rather than philosophical: a "resurrection" failure is a measurable event because the timeline includes the fact ID that was invalidated and the system's response either mentions it or doesn't.

## The harness — `MemoryStrategy` interface

To plug a system in, you implement (`README.md:267-291`):

```python
class MemoryStrategy:
    def process_event(self, event: Event) -> None: ...
    def build_context(self, query: Query, token_budget: int) -> str: ...
    def reset(self) -> None: ...
```

Then register in `src/statebench/baselines/__init__.py` (`CONTRIBUTING.md:14-18`). Run:

```bash
statebench evaluate -d data/benchmark.jsonl -b my_strategy -m gpt-5.2
```

This is the right interface. It refuses to test the memory store in isolation (which would let systems claim wins on a contrived API surface) and instead tests **how the strategy assembles context for an LLM under a fixed token budget**. To benchmark Mem0, you wrap `memory.search()` in `build_context`. To benchmark Volt, you wrap the active-context assembler. To benchmark Neo, you wrap `retrieve_relevant`. Same yardstick.

Two evaluation paths:

1. **Native Python harness** (`src/statebench/evaluation/`, `src/statebench/baselines/`). Judge uses gpt-4o-mini or claude-3-haiku for paraphrase detection (`docs/EVALUATION.md:124-155`).
2. **lm-eval-harness / Lighteval integration** (`lighteval_tasks/README.md:1-39`). Flattens timelines into single-query docs, pre-computes context via `transcript_replay` baseline.

## The ten reference baselines

(`README.md:232-246`)

| Baseline | What it is |
|---|---|
| **memgine** | The full deterministic state engine (query-relevance sort, engine-level access control, adaptive repair, threshold compaction) |
| `state_based` | Reference implementation of the spec without Memgine's four innovations |
| `state_based_no_supersession` | Ablation: state without supersession tracking |
| `fact_extraction_with_supersession` | Fact store + supersession (Mem0-style if Mem0 had supersession) |
| `fact_extraction` | Pure extraction (closest baseline to Mem0 OSS) |
| `rolling_summary` | LLM-summarized history |
| `rag_transcript` | Retrieved transcript chunks (vanilla RAG over conversation) |
| `transcript_replay` | Raw conversation history |
| `transcript_latest_wins` | Transcript with recency bias |
| `no_memory` | No history — control |

Read these together and the design space is fully mapped. Each one is an architectural bet; the benchmark scores tell you which bet pays off on which failure mode.

## Headline numbers from v1.0 dev split

(`README.md:53-98`)

| System / model | Decision accuracy | SFRR |
|---|---|---|
| memgine / Opus 4.6 | **97.3% ± 0.5%** | **37.1%** |
| best baseline (`state_based`) / Opus 4.6 | 62.9% | — |
| `state_based` / GPT-5.2 | 80.3% | — |
| `state_based` / Opus 4.5 | 62.9% | — |

Notable: GPT-5.2 beats Opus 4.5 by ~17 points on `state_based` — model capability matters even when memory architecture is held constant. Memgine adds ~17 more points on top of that. The split lets you isolate "what does the engine give you" from "what does the model give you."

## Submissions, official protocol, contamination

Submissions are CLI-generated (`README.md:186`):

```bash
statebench leaderboard --baseline memgine --submitter "YourOrg" --model gpt-5.2
```

The submission JSON (`submissions/submission.json:1-84`) records protocol version, submitter, benchmark release + split + dataset SHA-256, system config (baseline / model / provider), evaluation metadata (seeds, num_runs), and full results with mean/std across runs.

Official submissions require **3+ seeds minimum**, **cryptographic signing**, and an attestation that submissions **don't include test or hidden data** (`README.md:319-328`, `V1_SPEC.md:879-894`). The 10 canary items (`metadata.json:1-18`) are the contamination detection mechanism — if a model has seen them, they'll show up in outputs.

This is the production-grade benchmark hygiene a lot of newer evals skip. Worth noting if you're designing your own.

## What's inside this submodule

| Path | What's there |
|---|---|
| `README.md` | Overview, quick start, CLI, baseline table, headline numbers |
| `docs/EVALUATION.md` | Scoring rubrics, judge prompts, deterministic matching, multi-seed protocol |
| `docs/ALGORITHM.md` | Four-layer context specification (the state-based architecture itself) |
| `docs/V1_SPEC.md` | Timeline schema, detection requirements, provenance, versioning contract |
| `docs/MEMGINE.md` | How Memgine tests the paper's predictions |
| `docs/state-based-context-architecture.pdf` | Liotta 2025 paper |
| `docs/memgine-deterministic-memory-engine.pdf` | Liotta 2026 paper |
| `data/releases/v1.0/` | The 1,410-timeline dataset + metadata |
| `src/statebench/baselines/` | Ten reference strategies |
| `src/statebench/evaluation/` | Native Python harness |
| `lighteval_tasks/` | lm-eval-harness integration |
| `statebench-lm-eval/` | Additional lm-eval bridge |
| `submissions/` | Submission JSON examples |
| `results/` | Published results |
| `spaces/statebench-explorer/` | Gradio app for browsing timelines (`spaces/statebench-explorer/app.py:1-75`) |
| `docker/` | docker-compose with CPU/GPU profiles for sandboxed evaluation |
| `CONTRIBUTING.md` | How to add a baseline or a track |

If you only read three files: `README.md` (the framing), `docs/EVALUATION.md` (the scoring), `docs/V1_SPEC.md` (the timeline format you'll be implementing against).

## Mental model for using it well

- **Use the failure-mode taxonomy independently of running the benchmark.** Even just reading `README.md:105-155` and asking "which of these could happen to my agent" is half the value.
- **Run the ablations first.** `state_based` vs `state_based_no_supersession` is more informative than `memgine` vs `transcript_replay`. The ablation tells you which feature earned its keep.
- **Treat SFRR as the keystone metric.** A system can hit 90% Decision Accuracy by reciting everything it ever heard — at the cost of resurrecting half the invalidated facts. The Memgine 97.3% / 37.1% combination is the headline because both numbers are non-trivial.
- **Plug in via `MemoryStrategy`, not by porting to the harness's preferred shape.** The interface is small for a reason. Wrap your existing API rather than rewriting.
- **Respect the multi-seed protocol for any number you publish.** Single-seed scores are noise.
- **Don't let the model see the canary items.** If you're fine-tuning, the canaries are the easiest way to catch leakage.

## When NOT to reach for this

- **You're not building a stateful agent.** If your agent is single-turn or genuinely stateless, StateBench measures nothing relevant.
- **Your failure modes are domain-specific in ways the timeline format can't express.** Code-correctness failures, multimodal-grounding failures, tool-use failures — StateBench doesn't measure these.
- **You only care about retrieval quality, not state correctness.** LoCoMo / LongMemEval / BEAM (the benchmarks Mem0 cites) are more aligned with retrieval-quality framings.
- **You want a leaderboard you can climb publicly today.** The HuggingFace Space (`parslee/statebench-explorer`) is for browsing timelines, not a live scoreboard. The leaderboard is curated.

## What we learn from this entry being in the study

Three things:

1. **The failure-mode taxonomy is the artifact.** Before optimizing any memory system, decide which of the six failure modes you can tolerate and which you can't. Most production failures we've seen map cleanly onto one of the six.
2. **State-based architectures with supersession outperform fact-extraction architectures on the failure modes that matter most for trust.** Memgine vs `fact_extraction_with_supersession` is the comparison that motivates Neo's design and exposes Mem0's structural gap.
3. **Benchmark hygiene is rare and worth copying.** Multi-seed, signed submissions, canary contamination detection, deterministic+LLM hybrid scoring with asymmetric strictness on must-not-mention — if we ever build an internal benchmark for any other Lossless system, this is the template.

## How this fits with the other study entries

| | Mem0 | Neo | Volt | StateBench |
|---|---|---|---|---|
| **Resurrection resistance** | Weak (no supersession in OSS) | Strong (deterministic 0.85 supersession) | Strong (immutable log, summary nodes never falsify) | Measures it |
| **Hallucination resistance** | Medium (extracted-only) | Medium (typed facts) | Strong (verbatim log) | Measures it |
| **Scope leak resistance** | Strong (required scope) | Strong (auto-detected from git) | Medium (per-conversation) | Measures it |
| **Best benchmark target** | LoCoMo / LongMemEval | Outcome detection on real codebases | OOLONG long-context | Itself |

StateBench is the only entry that scores the others on the failure modes that actually break production. Use it to keep the marketing claims honest.

## How this compares to our own `context-vigilance` skill

`context-vigilance` has no concept of supersession beyond "humans edit markdown files." There's no resurrection-rate to measure because there's no automated recall surface — when an agent reads `context-v/` it reads whatever's there *now*, so a superseded blueprint is gone the moment a human deletes the section.

This is a feature, not a bug, for the human-curated case. It's also why `context-vigilance` is **not** a candidate to be benchmarked by StateBench: the surface area is "files an agent grep'd" rather than "memory the agent assembled into context." StateBench's failure modes assume an active recall mechanism worth measuring.

The right takeaway: if we ever build an active recall mechanism on top of `context-v/` (e.g., embedding the markdown and ranking it), StateBench's failure-mode taxonomy is the first thing we'd design against.

## One-line summary

> StateBench operationalizes "stateful agent correctness" into six failure modes and four metric families over 1,410 long-horizon timelines, ships ten reference baselines spanning the realistic design space, and gives the study its only independent yardstick for the implementation claims everyone else makes about themselves.
