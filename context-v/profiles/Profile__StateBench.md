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
site_uuid: 53f5c99b-83b9-45cf-be49-848708ae967a
hex_code: sk0h9w
date_authored_initial_draft: 2026-05-17
date_authored_current_draft: 2026-11-23
lede: >-
  A benchmark that audited itself: StateBench 2.0 found its own scorer flagged
  every correct "not Friday, it moved" answer as a resurrection, and retracted
  the numbers built on it.
summary: >-
  Profile of StateBench (Parslee) as pinned in the memory-layers-for-agents
  study. It is the independent harness that lets the study stop taking every
  other entry's self-reported numbers on faith. Covers the six failure modes
  (resurrection, hallucination, scope leak, stale reasoning, authority
  violation, temporal decay failure), the four metric families and their hybrid
  deterministic-plus-LLM-judge scoring with its deliberate asymmetry, the
  1,410-timeline dataset with authority levels and canary contamination items,
  the three-method MemoryStrategy plug-in interface, the ten reference baselines
  that between them map the whole design space, and the multi-seed signed
  submission protocol. The failure-mode taxonomy is worth using on its own,
  without ever running the benchmark; the submission hygiene is the template to
  copy for any internal eval.
publish: true
title: StateBench — Profile
date_modified: 2026-11-23
authors:
  - Michael Staton
status: Reference
type: Profiles
---

# StateBench — Profile

A profile of StateBench as it lives in this study (`studies/memory-layers-for-agents/statebench/`). Cites pinned paths so you can jump to source rather than trust paraphrase. Unlike the other three entries, StateBench is **not a memory implementation** — it is the yardstick. Read it alongside [`Profile__Mem0.md`](./Profile__Mem0.md), [`Profile__Neo.md`](./Profile__Neo.md), and [`Profile__Volt.md`](./Profile__Volt.md) — those are the systems StateBench was built to evaluate.

## TL;DR

StateBench is a **conformance test for stateful AI agents** that operationalizes the abstract goal "state correctness over time" into six concrete failure modes and four metric families, then ships ten reference implementations spanning the obvious memory architectures so newcomers can see which design choices break which failure mode.

The dataset is 1,410 long-horizon **timelines** — sequences of conversation turns, state writes (with explicit fact IDs, sources, authority levels), supersessions, environment signals, and a query with ground truth — split 60/15/15/10 train/dev/test/hidden with 10 canary items for contamination detection (`data/releases/v1.0/metadata.json:1-18`, `docs/V1_SPEC.md:68-159`). Scoring is **hybrid** (`docs/EVALUATION.md:10-14`): deterministic phrase matching for unambiguous constraints, LLM-as-judge for paraphrase detection, multi-seed protocol for official submissions.

The reference implementation that motivates the whole project is **Memgine** — Parslee's deterministic memory engine — which on Opus 4.6 / v1.0 dev split hits **97.3% decision accuracy** with **37.1% superseded-fact resurrection** vs `state_based`'s 62.9% accuracy (`README.md:102-118`). These are 1.x-scored numbers; the 2.0.0 release (2026-08-04) says they are not reproducible and that every SFRR figure in them is inflated roughly 2×. See the changes section below before quoting them. The benchmark exists in large part to show that the state-based architecture from Liotta 2025/2026 (the two papers under `docs/`) outperforms transcript-replay, RAG, summary, fact-extraction, and fact-extraction-with-supersession baselines on the failure modes that actually matter for production agents.

**Why this matters for the study:** every other entry here is an *implementation* with a self-reported benchmark. StateBench is the independent harness that lets you not trust those self-reports.

## The six failure modes

These are the framing concepts the whole benchmark organizes around (`README.md:154-208`):

| # | Failure mode | What it means |
|---|---|---|
| 1 | **Resurrection** | System references facts that were explicitly invalidated |
| 2 | **Hallucination** | System asserts state that was never established |
| 3 | **Scope leak** | Information crosses scope/tenant/session boundaries |
| 4 | **Stale reasoning** | System ignores corrections in downstream reasoning |
| 5 | **Authority violation** | Lower-authority sources override higher (peer overrides policy) |
| 6 | **Temporal decay failure** | Time-sensitive state treated as permanent |

This taxonomy is the most useful artifact StateBench produces, even before you run a single benchmark. It gives you a vocabulary for what your agent gets wrong, and it maps cleanly to the architectural choices in Mem0 (resurrection-exposed), Neo (resurrection-resistant via supersession), and Volt (hallucination-resistant via immutable log).

A central finding (`docs/EVALUATION.md:96-118`): systems that include **more context** achieve higher decision accuracy but **also higher resurrection rates** — a fundamental trade. Memgine's whole claim is that supersession-tracking + relevance filtering breaks the trade. That section is unchanged at the 2026-11-23 pin, but the project's own 2.0 audit (`docs/paper-measurement-validity.md` §6.3) shows much of the measured trade was a scoring artifact: context-rich baselines discussed the superseded value in order to reject it and were charged for it. Treat the trade as smaller than the 1.x numbers suggested.

## Metrics

Four families (`README.md:478-487`; 2.0 adds `leakage_rate` and `fabrication_rate` reported separately, and the README also lists False Refusal Rate):

- **SFRR** — Superseded Fact Resurrection Rate. Target 0%.
- **Decision Accuracy** — correct yes/no/value.
- **Must Mention Rate** — required phrases appear.
- **Must Not Mention Violation Rate** — forbidden phrases don't appear.

Scoring details (`docs/EVALUATION.md:25-94`):

```
SFRR = (queries with must_not_mention violations) / (queries with any must_not_mention constraints)
     # deterministic only via contains_phrase()
     # 2.0: counts resurrection only; boundary-aware matching; phrases under a
     # negation cue and non-discriminative phrases are excluded

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

Track categories (`README.md:242-261`, `V1_SPEC.md:775-828`): supersession (explicit and implicit), durability (commitments survive interruptions), access control (scope/authority/privacy), temporal (freshness), integrity (identity/contradiction), and an adversarial track that combines failure modes. Later additions: `premise_resistance` / `premise_maintain`, `deletion_compliance` / `deletion_maintain`, paired-counterfactual `cf_*` tracks, and an `applicability` factorial (`README.md:263-342`).

The dataset format makes the failure modes **operational** rather than philosophical: a "resurrection" failure is a measurable event because the timeline includes the fact ID that was invalidated and the system's response either mentions it or doesn't.

## The harness — `MemoryStrategy` interface

To plug a system in, you implement (`README.md:533-564`, `src/statebench/baselines/base.py:118-160`):

```python
class MemoryStrategy:
    def process_event(self, event: Event) -> None: ...
    def build_context(self, query: str) -> ContextResult: ...  # context + provenance sources
    def get_system_prompt(self) -> str: ...
    def reset(self) -> None: ...
```

The token budget is set by the harness (default 8K, `README.md:516`), not passed per call.

Then register in `src/statebench/baselines/__init__.py` (`CONTRIBUTING.md:14-18`). Run:

```bash
statebench evaluate -d data/benchmark.jsonl -b my_strategy -m gpt-5.2
```

This is the right interface. It refuses to test the memory store in isolation (which would let systems claim wins on a contrived API surface) and instead tests **how the strategy assembles context for an LLM under a fixed token budget**. To benchmark Mem0, you wrap `memory.search()` in `build_context`. To benchmark Volt, you wrap the active-context assembler. To benchmark Neo, you wrap `retrieve_relevant`. Same yardstick.

Two evaluation paths:

1. **Native Python harness** (`src/statebench/evaluation/`, `src/statebench/baselines/`). Judge uses gpt-4o-mini or claude-3-haiku for paraphrase detection (`docs/EVALUATION.md:120-156`; since 2.0 the judge is pinned globally via `judge_provider` / `judge_model` or `$STATEBENCH_JUDGE` instead of following the model under test).
2. **lm-eval-harness / Lighteval integration** (`lighteval_tasks/README.md:1-39`). Flattens timelines into single-query docs, pre-computes context via `transcript_replay` baseline.

## The ten reference baselines

(`README.md:499-516`)

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

The registry (`src/statebench/baselines/__init__.py`) also carries variants beyond the ten: `memgine_safe_supersession`, `memgine_distill`, `car_memgine` (a Rust engine), `dialectic`, `flat_file`, and three `reconstructive_rag_*` baselines added in 2.0. Read the ten together and the design space is fully mapped. Each one is an architectural bet; the benchmark scores tell you which bet pays off on which failure mode.

## Headline numbers from v1.0 dev split

(`README.md:55-152`; 1.x scoring, retained upstream as "the published record")

| System / model | Decision accuracy | SFRR |
|---|---|---|
| memgine / Opus 4.6 | **97.3% ± 0.5%** | **37.1%** |
| best baseline (`state_based`) / Opus 4.6 | 62.9% | — |
| `state_based` / GPT-5.2 | 80.3% | — |
| `state_based` / Opus 4.5 | 62.9% | — |

Notable: GPT-5.2 beats Opus 4.5 by ~17 points on `state_based` — model capability matters even when memory architecture is held constant. Memgine adds ~17 more points on top of that. The split lets you isolate "what does the engine give you" from "what does the model give you."

## Submissions, official protocol, contamination

Submissions are CLI-generated (`README.md:238-239`, `587-601`):

```bash
statebench leaderboard --baseline memgine --submitter "YourOrg" --model gpt-5.2
```

The submission JSON (`submissions/submission.json:1-84`) records protocol version, submitter, benchmark release + split + dataset SHA-256, system config (baseline / model / provider), evaluation metadata (seeds, num_runs), and full results with mean/std across runs.

Official submissions require **3+ seeds minimum**, **cryptographic signing**, and an attestation that submissions **don't include test or hidden data** (`README.md:587-601`, `V1_SPEC.md:879-894`). The 10 canary items (`metadata.json:1-18`) are the contamination detection mechanism — if a model has seen them, they'll show up in outputs.

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

- **Use the failure-mode taxonomy independently of running the benchmark.** Even just reading `README.md:154-208` and asking "which of these could happen to my agent" is half the value.
- **Run the ablations first.** `state_based` vs `state_based_no_supersession` is more informative than `memgine` vs `transcript_replay`. The ablation tells you which feature earned its keep.
- **Treat SFRR as the keystone metric.** A system can hit 90% Decision Accuracy by reciting everything it ever heard — at the cost of resurrecting half the invalidated facts. Under corrected 2.0 scoring the dev-split figures for Memgine on GPT-5.2 are 96.8% accuracy / 14.0% SFRR / 1.9% leakage (`docs/paper-measurement-validity.md` §6.4).
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

## Changes since the previous pin (2026-03-27 → 2026-10-07)

`5359ae4c120d` → `fe9207a16d72`, 72 commits. Releases in range: v2.0.0 (2026-08-04, `CHANGELOG.md`); the README and `docs/EVALUATION.md` label later additions "v2.1" but `pyproject.toml` still says 2.0.0 and there is no v2.1 tag. License unchanged (MIT, Parslee, LLC).

- **The scorer was wrong, and the project says so.** 2.0.0 fixes six defects in the evaluation instrument (`CHANGELOG.md`, `docs/paper-measurement-validity.md`). The one that matters: phrase-list scoring could not tell naming a superseded value to reject it from asserting it, so 100% of correct answers phrased as rejections were scored as resurrections. Fixes: a globally pinned judge, boundary-aware phrase matching, exclusion of non-discriminative forbidden phrases, negation awareness, SFRR counting resurrection only (`leakage_rate` and `fabrication_rate` split out), and word-boundary decision extraction. 1.x and 2.x numbers are not comparable.
- **Headline claims moved.** The README leaderboard (`README.md:55-101`) is kept as the published record with a warning that SFRR figures are inflated about 2× and the baseline ordering changes. Under corrected scoring Memgine falls from 4th to 9th of ten on SFRR (§6.3), and the Memgine paper's claim of better accuracy "without increasing leakage" is withdrawn; its accuracy lead and its 1.9% leakage rate hold (§6.4). A `gpt-5.6-sol` refresh narrows Memgine's dev-split lead over `state_based` from 9.1pp to 0.7pp (§6.5), a result the README scopes to short dependency distances.
- **The generator was audited too.** `statebench audit-dataset` (`README.md:343-400`, `src/statebench/generator/fidelity.py`) found 105 out-of-order event errors and 2 unsatisfiable queries in the shipped v1.0 release; newly generated data audits clean. Dependency distance is now a generation parameter with `statebench distance-sweep` (`src/statebench/generator/distance.py`).
- **New tracks and baselines.** `premise_resistance`, `deletion_compliance`, paired-counterfactual `cf_*` governance tracks, an `applicability` factorial, Governance Bypass Rate and Unsupported Reconstruction Rate scored from the engine's audit record, `reconstructive_rag_*` baselines, and `statebench.skills` for distilling procedures from frontier-model episodes (`src/statebench/skills/`). Memgine gained authority-aware same-key conflict resolution, transitive supersession propagation, and a Rust `car_memgine` arm.
- **Three draft papers** under `docs/` (`paper-measurement-validity`, `paper3-retrieval-reconstruction-governance`, `paper4-worked-examples`) and a review of a 2026 agent-memory survey (`docs/research-review-2026-08-agent-memory.md`). The README marks the drafts as superseding the two published papers where they disagree.

For this study the self-correction strengthens the entry's role as the independent yardstick: it is the only entry that has published a retraction of its own numbers.

## One-line summary

> StateBench operationalizes "stateful agent correctness" into six failure modes and four metric families over 1,410 long-horizon timelines, ships ten reference baselines spanning the realistic design space, and gives the study its only independent yardstick for the implementation claims everyone else makes about themselves.
