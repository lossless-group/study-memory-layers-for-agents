---
name: Delta-Mem Profile
slug: delta-mem
upstream: https://github.com/declare-lab/delta-Mem
package: deltamem (HF adapter: declare-lab/delta-mem_qwen3_4b-instruct)
license: CC-BY-4.0
maintainer: declare-lab (SUTD) — Jingdi Lei, Di Zhang, Junxian Li, Weida Wang, Kaixuan Fan, Xiang Liu, Qihan Liu, Xiaoteng Ma, Baian Chen, Soujanya Poria
study: studies/memory-layers-for-agents
profile_path: studies/memory-layers-for-agents/delta-mem
profile_kind: research-artifact + transformer-adapter + training-pipeline
date_created: 2026-05-17
---

# Delta-Mem — Profile

A profile of Delta-Mem as it lives in this study (`studies/memory-layers-for-agents/delta-mem/`). Cites pinned paths so you can jump to source rather than trust paraphrase. **Read this one differently from the others.** Every other entry in the study (Mem0, MemPalace, Graphiti, Neo, Volt) is an *external* memory layer the agent talks to. Delta-Mem is **memory baked into the transformer itself** — a learned adapter that modifies attention computation so the model carries an updatable state through inference. That makes it the most architecturally distinct entry and the only one that is a *research artifact* rather than a deployed product.

## TL;DR

Delta-Mem is the official code for the paper [*δ-mem: Efficient Online Memory for Large Language Models*](https://arxiv.org/abs/2605.12357) (declare-lab / SUTD, May 2026, `README.md:6`). The thesis: external retrieval pays unbounded context cost and static parametric memory is frozen at training time. What's needed is **a compact Online State of Associative Memory alongside a frozen full-attention backbone** that "updates dynamically during interaction and directly influences the model's internal computation during inference" (`README.md:17, 23`).

Mechanically, Delta-Mem wraps each attention head with a `DeltaMemAttention` module (`deltamem/delta_impl.py:500`) that projects hidden states into a low-rank (default rank=8) memory space and maintains a dense **state matrix** `delta_state` (`delta_impl.py:614`) per head per batch, updated token-by-token via a **delta-rule** affine scan (`delta_impl.py:1927`):

```
S_{t+1} = λ_t · S_t  −  β_t · (S_t k_t) ⊗ k_t  +  β_t · v_t ⊗ k_t
```

where λ (keep) and β (write/erase) are learned per-token gates. Three write granularities — **TSW** (token), **SSW** (sentence), **MSW** (message) — control how often the state is updated (`README.md:19`, `delta_impl.py:40-44`). The released model uses TSW + rank-8 Q/O + write length 8192 on Qwen3-4B-Instruct (`README.md:29-30`, available at `declare-lab/delta-mem_qwen3_4b-instruct`).

Critically: the base model weights are **frozen**. Only the adapter (the delta-rule projections + gates) is trained. But this is **not** a standard PEFT LoRA — adapters are *not* mergeable (`README.md:214`) because the runtime memory read/write path is part of model execution.

If you want one sentence: **Delta-Mem is a frozen-backbone transformer adapter that gives each attention head a low-rank dense state matrix updated by a learned delta rule, making memory an architectural property of the model rather than an external system the agent queries.**

## Why this is in a study of "memory layers for *agents*"

This is the framing question and worth being explicit about. Every other entry in the study answers "where does the agent store and look things up?" Delta-Mem answers "what if the *model* never forgot in the first place, because state is part of its forward pass?"

Two reasons it belongs:

1. **It's the architectural alternative the system-level entries are implicitly betting against.** Mem0 and Graphiti and MemPalace all assume the model is a black box with a context window and you have to feed it the right things. Delta-Mem says: maybe the right answer is to change the model. Reading this profile next to the others makes the design space legible.
2. **It produces a benchmark contender.** Delta-Mem is evaluated on **LoCoMo** (`eval/locomo_delta.py`, `eval/locomo_protocol.py`) — the same benchmark Mem0 ships SOTA numbers on. The comparison isn't apples-to-apples (different models, different protocols) but it is the only entry in the study that competes for the same scoreboard from a structurally different direction.

If you build agent systems, you probably won't deploy Delta-Mem directly — it requires retraining adapters on the base model you ship. But the *idea* — "what if memory was a forward-pass artifact, not a retrieval-and-reinject artifact" — is the one external memory architectures don't have an answer to.

## The architectural design

Delta-Mem replaces each `self_attn` module in target transformer layers (`deltamem/delta.py:94`, `delta_impl.py:500`). Inside the wrapped attention:

**Six new linear projections** (`delta_impl.py:590-597`):

- `memory_q_proj`, `memory_k_proj`, `memory_v_proj` — project hidden states into a low-rank memory space of size `state_read_dim = rank × num_state_heads`.
- `delta_q_proj`, `delta_k_proj`, `delta_v_proj`, `delta_o_proj` — project the memory readout back to standard QKV/O space so it composes with the frozen base attention.

**The state matrix** (`delta_impl.py:614`): a square `rank × rank` tensor per attention head, maintained per batch element across the sequence. This is what makes Delta-Mem *stateful* in a way standard transformers aren't.

**Update rule** (`delta_impl.py:1927`):

```python
next_state = keep_t * current_state - erase_t * pred_outer + write_t * write_outer
```

`keep_t` is the λ gate (controls how much of the old state survives); `erase_t` and `write_t` derive from the β gate (control overwriting and adding the new outer product). Each update is a **rank-r outer product** — efficient, parallelizable, and the closest analogue to a classic "delta rule" from associative-memory literature.

Hyperparameters (`delta_impl.py:530`): default `rank=8`, `alpha=16.0` (so `delta_scaling = alpha/rank = 2`), plus initialisation values for the β/λ biases. Optional `rankwise_gates=True` makes the gates per-rank-dimension rather than scalar.

The state can be **read/written across sessions** via `get_delta_mem_online_state` / `load_delta_mem_online_state` (`delta_impl.py:2727, 2736`). This is the mechanism that turns the adapter into a persistent-memory device — the state lives outside the conversation and can be saved, restored, transferred.

## Three write granularities

(`README.md:19`, `delta_impl.py:40-44, 1417-1527`, `deltamem/.../write_segmentation.py:31`)

| Mode | When state updates | Use case |
|---|---|---|
| **TSW** — Token Segment Write | Every token | Most reactive; default for the released model |
| **SSW** — Sentence Segment Write | At sentence boundaries (hidden states accumulated then written) | Coarser, lower update cost |
| **MSW** — Message Segment Write | At message boundaries (via `message_ids`) | Coarsest; one state update per dialogue turn |

This is the only entry in the study where "write policy" is a *temporal granularity choice for a learned mechanism*, not a "do we LLM-extract or store verbatim" choice. Different question entirely.

## The kernel — Triton-accelerated affine scan

The state update is a recurrence (each step depends on the previous), so it can't be trivially batched. Delta-Mem implements it as an **affine scan** with a Triton kernel (`deltamem/.../affine_scan.py`, called from `delta_impl.py:1895-1938`) plus a pure-PyTorch fallback. CUDA-recommended; CPU-only use isn't supported in practice (`README.md:70`).

This is also where the "efficient" in "Efficient Online Memory" earns its keep — the scan is sub-quadratic in sequence length, unlike full-attention memory layouts that scale O(n²).

## Training pipeline

`deltamem/train/` (`delta_sft.py`, `delta_sft_experimental.py`):

- **SFT-based supervised fine-tuning** on dialogue/episode data with standard causal-LM loss.
- **DeepSpeed ZeRO-2** for distributed training, bf16 precision (`deepspeed_zero2.json`).
- Pre-tokenized dataset caching supported (`train/delta_sft.py:25-27`).
- Scripts (`scripts/run_qasper_multimodel_write8192_train_and_benchmark_suite.sh`) automate full training + evaluation suites across TSW/SSW/MSW variants on QASPER.
- **Base models tested:** Qwen3-4B/8B and SmolLM3-3B (`README.md:29`).

Only the adapter parameters train. The frozen-backbone choice keeps cost bounded and means an adapter for a given base model is portable across deployments — but it also means **you can only Delta-Mem-ify a model you have weights for and can adapter-train**.

## Evaluation suite

(`README.md:263-286`, `deltamem/eval/`)

| Benchmark | What it tests | Module |
|---|---|---|
| **LoCoMo** | Multi-hop reasoning + temporal grounding over long conversations | `locomo_delta.py`, `locomo_protocol.py` |
| **HotpotQA** | Multi-hop QA | `benchmark_compare.py --tasks hotpotqa` |
| **IFEval** | Instruction-following | `benchmark_compare.py --tasks ifeval` |
| **GPQA Diamond** | Graduate-level adversarial QA | `benchmark_compare.py --tasks gpqa_diamond` |
| **MemoryAgentBench** | Agent-memory benchmarks | `official_memory_agent_bench.py` |

The LoCoMo protocol (`locomo_protocol.py:21-48`) tests multi-hop, temporal, single-hop, and adversarial categories with F1/EM after conversation-history replay. Crucially, it tests whether memory **persists and updates correctly across turns** — exactly the property an external retrieval system would fail at differently.

Numbers and direct comparisons against Mem0 / MemPalace on LoCoMo would require running each system through the other's harness. The paper presumably reports head-to-head numbers; the code here is what would let you reproduce them.

## What's inside this submodule

| Path | What's there |
|---|---|
| `deltamem/delta.py` | `attach_delta_mem(model, config)` — the entry point that wraps target attention modules |
| `deltamem/delta_impl.py` | The core: `DeltaMemAttention`, the state-matrix logic, the delta-rule scan (~2700 LOC) |
| `deltamem/.../affine_scan.py` | Triton kernel for the sub-quadratic scan + PyTorch fallback |
| `deltamem/.../write_segmentation.py` | Sentence/message segmentation for SSW/MSW modes |
| `deltamem/.../backbone_compat.py` | Compatibility shims for Qwen3Attention / SmolLM3Attention |
| `deltamem/train/delta_sft.py` | Supervised fine-tuning entry point |
| `deltamem/runtime/session.py` | `DeltaMemChatSession` — session wrapper, state reset, message-ID tracking |
| `deltamem/runtime/chat_cli.py` | Minimal interactive demo |
| `deltamem/eval/` | LoCoMo, HotpotQA, IFEval, GPQA Diamond, MemoryAgentBench harnesses |
| `data/` | Training/eval data (QASPER, LoCoMo, etc.) |
| `scripts/` | End-to-end train-and-benchmark shell scripts |
| `deepspeed_zero2.json` | DeepSpeed config |
| `requirements.txt` | Python deps |

If you read three files: `README.md` (the framing and the API), `deltamem/delta_impl.py` (the architecture, especially the `DeltaMemAttention` class and the scan around line 1895-1938), `deltamem/eval/locomo_protocol.py` (how memory persistence is operationalized as a test).

## Public release

Hugging Face: [`declare-lab/delta-mem_qwen3_4b-instruct`](https://huggingface.co/declare-lab/delta-mem_qwen3_4b-instruct) (`README.md:12`) — the TSW adapter for Qwen3-4B-Instruct.

Loading (`README.md:192-212`):

```python
config = HFDeltaMemConfig.from_pretrained(adapter_dir)
attach_delta_mem(model, config)
load_delta_mem_adapter(model, adapter_dir)
```

Inference requires running the wrapped model with proper session management — the per-session state (`delta_state`) is not pre-populated; you reset/save/load it via the runtime APIs. This is closer to "manage a stateful service" than to "call a stateless library."

## Mental model for using it well

- **Treat it as an architectural experiment, not a drop-in library.** If you're not training adapters on your own base model, you're using the released Qwen3-4B-Instruct + TSW combination as-is.
- **Pick the granularity for your use case.** TSW for high-fidelity, low-coarseness updates. MSW if you're processing long structured dialogues and a one-update-per-turn coarseness is acceptable.
- **Manage state explicitly.** No automatic per-user persistence. The state is yours to save, restore, and reset. This is more like managing a database connection than calling an SDK.
- **Don't expect to merge adapters into the backbone.** Delta-Mem isn't LoRA. The forward-pass code path is the integration; you ship the adapter alongside the backbone (`README.md:214`).
- **Benchmark on your domain before adopting.** The published LoCoMo wins are for Qwen3-4B with this specific adapter. Your domain, your base model, your evaluation.

## When NOT to reach for this

- **You're shipping an agent that uses a closed model (GPT, Claude, Gemini).** Delta-Mem requires base-model weights. It's structurally inapplicable to API-only models.
- **You don't have GPU infrastructure for training and inference.** This is research-grade CUDA code. CPU-only is not a path.
- **You want a stable production library with backward-compatibility guarantees.** This is a research artifact. The API is the code.
- **Your memory problem is "give the agent persistent facts about Alice across sessions."** That's a retrieval problem (Mem0, Graphiti, MemPalace). Delta-Mem can do it in principle by persisting state across sessions, but it's not what the design optimises for.
- **You want explicit, auditable memory records.** Delta-Mem's state is a dense numeric matrix. There's no `cat memory.json` here. If audit matters, an external system is the right call.

## How this compares to the rest of the study

| Axis | Delta-Mem | Mem0 / MemPalace / Graphiti / Neo / Volt |
|---|---|---|
| **Where memory lives** | Inside the model (per-head state matrix) | Outside the model (vector store / graph DB / Postgres / JSON) |
| **What's stored** | Dense low-rank numeric state | Text, embeddings, edges, summaries |
| **Update mechanism** | Learned delta rule, per token | LLM extraction, supersession, or verbatim append |
| **Provenance** | None (state is opaque) | First-class (every other entry preserves source) |
| **Audit** | Numeric state, not human-readable | All others are inspectable |
| **Cross-model portability** | None (adapter is per-base-model) | All others are model-agnostic |
| **Required infrastructure** | GPU + base model weights + training run | A database; usually a CPU |
| **Failure mode under contradiction** | Latest tokens dominate via λ decay | Depends on the system (supersession in Neo, temporal in Graphiti, none in Mem0/MemPalace) |
| **Best fit** | Research; long-context single-model scenarios where you control the stack end-to-end | Agent products in any LLM stack |

The key insight: **Delta-Mem and the external-memory systems aren't competitors — they're stacked layers of the same problem**. You could in principle ship a Delta-Mem-equipped backbone *and* a Mem0 retrieval layer. Delta-Mem would carry rolling state through the immediate conversation; Mem0 would surface durable facts from prior sessions. They optimise different time scales.

The study is more complete with Delta-Mem in it because it forces the question "is this even a retrieval problem?" — which the other five entries quietly assume.

## How this compares to our own `context-vigilance` skill

These two are at *maximum distance* on every axis the study cares about. `context-vigilance` is markdown files reviewed in PRs by humans, with structure as the value. Delta-Mem is a dense numeric state matrix updated by a learned rule inside a transformer's forward pass.

There's no composition story; they don't speak the same vocabulary. The honest mapping: `context-vigilance` is *durable, human-readable, agent-grep-able* project knowledge; Delta-Mem is *transient, opaque, model-internal* working memory. Both real, both useful, both at opposite ends of the spectrum the study spans.

The transferable lesson from Delta-Mem back to our work isn't a code pattern — it's the *framing*. When someone asks "how should the agent remember X," the first question to ask is "should this be retrieved or should this be in the model's working state?" `context-vigilance` is retrieval. Delta-Mem is working state. Knowing the difference is the lesson.

## A note on license and provenance

- **License:** CC-BY-4.0 (`README.md:6`). Permissive with attribution. No LICENSE file at the repo root — this is documented in the README, which is sufficient but worth noting.
- **Affiliation:** declare-lab is the Singapore University of Technology and Design's NLP lab, the home of MELD, M2H2, Phi/MosaicML eval work, and multiple long-context-memory papers. The institutional credibility is real.
- **arXiv:** [2605.12357](https://arxiv.org/abs/2605.12357), May 2026.

## One-line summary

> Delta-Mem is the only entry in the study that puts memory *inside* the model — a frozen-backbone transformer adapter that gives each attention head a low-rank dense state matrix updated by a learned delta rule, with three temporal write granularities (TSW/SSW/MSW), a Triton-accelerated affine scan, and a public Qwen3-4B-Instruct adapter — making it the architectural alternative the system-level entries are implicitly betting against, and the right reference point for any conversation about whether agent memory is fundamentally a retrieval problem or a forward-pass problem.
