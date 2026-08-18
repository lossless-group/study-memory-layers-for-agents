---
name: Paxm Profile
slug: paxm
upstream: https://github.com/pax-beehive/paxm
package: paxm (Go module github.com/pax-beehive/paxm)
license: Apache-2.0
maintainer: pax-beehive
study: studies/memory-layers-for-agents
profile_path: studies/memory-layers-for-agents/paxm
profile_kind: cli + mcp-server + multi-agent-plugin + durable-write-queue
date_created: 2026-07-20
site_uuid: 7daa2f65-be10-40e4-a932-e3abf998cd0a
hex_code: vdfojd
date_authored_initial_draft: 2026-07-20
date_authored_current_draft: 2026-07-20
lede: >-
  Paxm is not a memory store — it is routing and durability in front of
  seven-plus pluggable backends, and it does zero semantic dedup.
summary: >-
  Profile of Paxm (pax-beehive) as pinned in the memory-layers-for-agents study.
  It is the study's clearest example of memory-as-plumbing rather than
  memory-as-store, and has the widest agent-harness integration of any entry
  (Claude Code, Codex, OpenCode, Pi, plus MCP). Covers the MemoryItem schema and
  its Origin-versus-Scope split (attribution is not authorization), the
  durable capture queue with its events/episodes/deliveries tables and
  crash-recovery path, per-provider goroutine bulkheads, the ranking calibration
  and its deterministic tie-breaking, exact-fingerprint LTM admission, the
  tier-plus-expiry eviction story, and the passive/active recall split with its
  first-turn-versus-later-turn asymmetry. Read docs/architecture.md upstream
  before the code — this profile names it as the best architecture doc in the
  study.
publish: true
---

# Paxm — Profile

A profile of Paxm as it lives in this study (`studies/memory-layers-for-agents/paxm/`). Cites pinned paths so you can jump to source rather than trust paraphrase. Read alongside [`Profile__Mem0.md`](./Profile__Mem0.md) for the general-memory-layer contrast and [`Profile__Volt.md`](./Profile__Volt.md) for the other durable-append-log architecture in the study.

## TL;DR

Paxm is not a memory store. It is a **routing and durability layer that sits in front of memory stores** — SQLite by default, or Zep/Mem0/Mem0-Cloud/MemOS/MemOS-Cloud/OpenViking/a custom JSON-RPC plugin as swappable backends (`docs/architecture.md:67-87`). The interesting engineering is almost entirely in two places: (1) a **provider-agnostic ranking calibration** that lets the router merge hits from structurally different scoring systems (BM25, cosine similarity, vendor-opaque ranks) without pretending they're comparable, and (2) a **durable local write queue** (SQLite WAL, checksummed, session-partitioned) that decouples "the agent hook returned" from "the memory provider received the write" — so a slow or down remote provider never blocks or loses a turn.

Where Neo (`Profile__Neo.md`) makes memory *the reasoning engine* and does deterministic cosine-threshold supersession, Paxm makes memory *a pluggable utility* and does no semantic reconciliation at all — writes are either exact-canonical-match consolidation (SHA-256 fingerprint) or they're just new rows. That's a deliberate scope boundary, not an oversight (`docs/architecture.md:239-243`: "The admission module only consolidates exact canonical matches. It does not use an LLM, infer semantic equivalence, resolve conflicting facts, promote STM, or supersede another memory.").

## Why this exists — the design bet

Paxm's bet is **integration breadth over storage sophistication**. It ships first-class hook integrations for four different coding-agent harnesses — Claude Code (5 native hooks), Codex, OpenCode, Pi — plus a stdio MCP server exposing `paxm_recall`, `paxm_remember`, `paxm_history`, `paxm_config_doctor` (`docs/architecture.md:191-196`, `internal/mcp/server.go:226,244,256,268`). No other entry in this study integrates with this many distinct agent harnesses at the hook level. The corollary bet is that **storage should be someone else's problem**: rather than building a best-in-class vector/graph store, Paxm defines a provider contract (`internal/memory/types.go:189-224`) and a shared black-box conformance test harness (`internal/adapters/contracttest`, referenced in `docs/provider-adapter-contract.md:3-6`) so seven+ backends — including one you write yourself over JSON-RPC — all satisfy the same guarantees.

The two bets compound: because the harness integrations are the hard-won part, and because a slow remote provider (Zep, Mem0 Cloud) would otherwise stall an agent's turn, the write path had to be made durable and asynchronous. That's why the capture queue (below) is arguably the most load-bearing subsystem in the codebase, not a side detail.

## The memory record schema

```go
type MemoryItem struct {
    ID            string
    Text          string
    AdmissionText string       // stable text used for LTM fingerprinting; may differ from Text
    Source        string
    Metadata      map[string]string
    CreatedAt     time.Time
    Tier          MemoryTier   // "stm" | "ltm"
    ExpiresAt     *time.Time
    Turn          *TurnContext // session_id, turn_id, started_at, ended_at
    Origin        MemoryOrigin // user_id, agent_id, session_id, turn_id
    Scope         MemoryScope  // type, id — the visibility boundary
    Provenance    Provenance   // v1-compat shape combining origin+scope
}
```
(`internal/memory/types.go:72-119`)

The **Origin vs. Scope split** is the schema's one real design opinion: `Origin` says *who produced this memory* (trusted runtime context, never caller-settable), `Scope` says *who can see it* (personal/team visibility boundary). Older integrations conflated these into one `Provenance` object, which is retained for compatibility but deprecated (`docs/provider-adapter-contract.md:28-54`). The doc is explicit that attribution is not authorization: "Attribution describes a stored memory; it is not proof of the current caller's identity and it does not grant access... An agent must not be able to widen recall by supplying these metadata keys" (`provider-adapter-contract.md:42-45`).

For providers that only support string metadata (most of them), the router mirrors structured `Origin`/`Scope` onto canonical keys — `paxm_user_id`, `paxm_agent_id`, `paxm_session_id`, `paxm_turn_id`, `paxm_scope_type`, `paxm_scope_id` (`internal/memory/lifecycle.go:12-21`) — on write via `PrepareProviderItem` (`lifecycle.go:46-68`), and reconstructs it on read via `ApplyHitAttribution` (`lifecycle.go:72-113`), falling back through legacy `Provenance` and then metadata in that order.

Compare to Neo (`Profile__Neo.md` §schema): Neo's `Fact` carries supersession pointers, dependency edges, confidence, and success counts — a record designed to be *reasoned over*. Paxm's `MemoryItem` carries almost none of that; it's designed to be *routed and delivered*, with the interesting state (rank, calibration) computed transiently at query time rather than stored.

## Write policy — durable queue, not durable store

Paxm's own writes are not directly synchronous provider calls. A hook event becomes a `capturequeue.Event`, appended to a local SQLite-WAL-backed queue (`internal/capturequeue/queue.go:37-44`, migration at `queue.go:887-935`):

```sql
CREATE TABLE capture_events (
  event_id, session_key, sequence, source_sequence, final_sequence,
  terminal, payload_json, payload_hash, episode_id, created_at,
  UNIQUE(session_key, sequence)
)
CREATE TABLE capture_episodes (
  episode_id, session_key, first_sequence, last_sequence, complete,
  payload_json, payload_hash, state, created_at,
  UNIQUE(session_key, first_sequence, last_sequence)
)
CREATE TABLE capture_deliveries (
  episode_id, provider, profiles_json, state, attempts,
  next_attempt_at, lease_until, last_error, provider_ref, delivered_at,
  PRIMARY KEY(episode_id, provider)
)
```
(`internal/capturequeue/queue.go:891-929`)

The flow, end to end:

1. **Append.** Each hook event is written with a SHA-256 `payload_hash` (`queue.go:1110-1112`, called at `queue.go:308,396-397`). The hook caller (`capture.Runtime.Process`, `internal/capture/runtime.go:344-378`) blocks only on this local transaction commit — `r.queue.Append(...)` then `r.notifyDelivery()` and returns immediately.
2. **Seal.** `turn_end` seals only its own session's pending events into one `Episode`, checking for sequence gaps (`Missing []int64`) and computing a whole-episode checksum over the joined per-event hashes (`queue.go:439`, verified again on read at `queue.go:1071-1084`). Sessions that never emit `turn_end` are sealed as incomplete after `capture_queue.max_episode_age` (`docs/architecture.md:359-360`, `runtime.go:86-89`).
3. **Deliver.** A background `worker` (`internal/capture/runtime.go:229-303`) wakes on a notify channel or a 1-second ticker, calls `queue.RunOnce`, and on failure calls `queue.RecoverDelivering` to reclaim any deliveries stuck mid-flight from a crash (`runtime.go:265-280`). SQLite defaults to 1 concurrent delivery worker; network providers default to 4 (`docs/architecture.md:373`, wired via `providerConcurrency` at `runtime.go:127-134`).
4. **Group by write profile, not 1:1.** `Episode.IngestInputs()` groups an episode's events by `(profile, tier, expires)` before delivery, so one episode can produce multiple provider-bound items if different events in the same turn used different write profiles (`queue.go:77-108`).

This is architecturally close to Volt's immutable-log approach (`Profile__Volt.md`) but scoped differently: Volt's log *is* the memory store; Paxm's queue is a **transport buffer in front of** whatever memory store you configured — durability of delivery, not durability of recall.

## Ranking — squared reciprocal rank, not shared score space

The router's central problem: SQLite returns a normalized BM25-derived relevance, Zep returns a graph-search score, Mem0 can return either similarity or cosine *distance* depending on deployment (`docs/provider-adapter-contract.md:60-80`) — none of these numbers mean the same thing. Paxm's answer is `calibrateProviderHits` (`internal/memory/ranking.go:10-31`):

```go
sort hits by normalized relevance, descending
for i, hit := range sorted:
    rankPrior := 1 / (rank² )       // rank = i+1
    hit.rankingScore = relevance * rankPrior
```

Per-provider hits are ranked *within that provider's own result set* first, then the squared-reciprocal-rank prior is applied — so a provider's #1 hit keeps most of its relevance weight, but its #10 hit is discounted by 1/100 regardless of what raw score it carried. The comment is explicit about what this is not: "Native score magnitudes are not treated as calibrated probabilities, and no candidate is promoted above the evidence supplied by its adapter" (`ranking.go:6-9`). Final ordering also folds in a weight (per-provider-route multiplier) and a recency boost that decays as `boost / (1 + age_hours/24)` (`router.go:596-605`), then breaks ties deterministically by score → relevance → createdAt → provider name → ID (`router.go:349-366`) — this determinism matters because concurrent provider completion order must never change which hit wins a dedup collision.

Deduplication (`dedupeSearchHits`, `router.go:333-347`) keys on scope-qualified normalized text when text is present, otherwise `provider:id` — so identical text in different scopes is deliberately kept as separate hits, matching the doc's claim that "identical text from different scopes is kept separate during recall deduplication" (`docs/architecture.md:219-220`).

Concurrency-wise, every provider is searched/written in its own goroutine (`searchProviders`/`putProviders`, `router.go:179-189, 427-437`) with a per-binding single-slot "bulkhead" channel (`runSearch`/`runPut`, `router.go:191-221, 439-465`) — a stuck provider occupies its own slot and its own timeout, not a shared pool, so one bad provider can't starve the others.

## Write-side admission — exact-match consolidation, deliberately not semantic

For long-term memory items written without an explicit ID, `admitLongTermMemories` (`internal/memory/lifecycle.go:190-218`) computes a SHA-256 fingerprint over canonicalized text (lowercased, whitespace-collapsed) plus workspace and scope, and assigns `ID = "ltm_" + fingerprint` (`lifecycle.go:212, 220-227`). SQLite then consolidates identical IDs within a transaction, bumping an occurrence counter and advancing `last_seen_at` while preserving the original `first_seen_at` (`docs/architecture.md:222-230`).

This is a real scope boundary worth naming precisely: it is **string-exact dedup**, not fuzzy/semantic dedup. Two paraphrases of the same fact get two different fingerprints and two different rows. Compare directly to Neo's cosine>0.85 supersession (`Profile__Neo.md` §write-policy) or RetainDB's bi-temporal validity — Paxm has no equivalent. The `AdmissionText`/`Text` split exists specifically to keep volatile fields (session IDs, timestamps embedded in a rendered hook template) out of the fingerprint identity for `user_input` events, while still storing the full rendered text as evidence (`docs/architecture.md:232-237`).

## Eviction — tier + expiry only

There is no scoring-based eviction, no LLM summarization pass, no consolidation-by-similarity. The entire eviction story is: `stm` items require a positive expiry (default 24h) and get deleted once expired; `ltm` items are rejected if configured with an expiry at all (`docs/architecture.md:212-214`). Cleanup is provider-opt-in via the `CleanupExpiredProvider` interface (`internal/memory/types.go:206-208`); SQLite implements it as a bounded batch delete of expired rows, scheduled as a best-effort background job after a successful hook flush and coalesced if already pending (`docs/architecture.md:457-465`). Recall correctness never depends on this running — both SQLite and the router filter expired hits at query time regardless (`hitMatchesPolicy`, `router.go:520-535`; `EffectiveHitExpiresAt`, `lifecycle.go:52-70`).

This is the leanest eviction story in the study — a deliberate consequence of not owning long-term storage semantics. Compare Neo's four coordinated eviction mechanisms (`Profile__Neo.md` §eviction) or Beads' Haiku-summarized "memory decay."

## Recall surface — three entry points, one policy

- **CLI:** `paxm recall --query TEXT [--limit N] [--json]`, `paxm remember [--profile stm|ltm] --text TEXT` (`docs/architecture.md:274-289`).
- **MCP:** `paxm mcp serve --agent NAME` exposes `paxm_recall`, `paxm_remember`, `paxm_history`, `paxm_config_doctor` over stdio (`internal/mcp/server.go:226,244,256,268`, dispatch at `server.go:321-327`) — "It reuses the same least-privilege tools and telemetry paths as the CLI, so recall/write policy remains entirely in user-owned paxm config" (`docs/architecture.md:191-196`). Notably, setup/uninstall/hook-install/routing/backfill are **not** MCP tools — an agent using MCP cannot reconfigure its own memory policy.
- **Passive hooks:** two distinct recall profiles depending on turn position — `passive_initial` (looser, RAG-warmup-shaped, used only for the first `user_input` in a session) and `passive` (limit 2, stricter thresholds, `ltm`-only) for every subsequent turn (`docs/architecture.md:175-180`). Active agent-driven recall via CLI/MCP defaults to reading both `stm` and `ltm` (`architecture.md:188-189`), so short-lived working memory can surface during active reasoning without ever being pushed passively into context.

That passive/active split — and specifically the *first-turn-vs-later-turn* asymmetry inside passive recall — doesn't have a direct analog elsewhere in this study; most other entries expose one recall API and leave call-site policy to the caller.

## Operational story — one binary, four harness integrations

| Entry point | Harness | Hook coverage |
|---|---|---|
| Native plugin, 5 hooks | Claude Code | `SessionStart`, `UserPromptSubmit`, `PostToolUse`, `PostToolUseFailure`, `Stop` (`docs/architecture.md:260-268`) |
| Native plugin | Codex | `SessionStart`, `UserPromptSubmit`, `Stop`; transcript-derived tool capture at `turn_end` since Codex has no native `PostToolUse` (`architecture.md:329-333`) |
| Extension system | Pi | `before_agent_start`, `message_end`, `tool_execution_start/end`, `agent_end`, `session_shutdown` (`architecture.md:467-476`) — event-bus-based, explicitly "best-effort" (`architecture.md:487-489`) |
| Lazy bootstrap | OpenCode | Same `SessionStart`/`UserPromptSubmit`/`Stop` shape, bootstrapped before first chat message since OpenCode doesn't expose a native session-start hook (`architecture.md:350-352`) |
| stdio | MCP clients generally | `paxm mcp serve` |

Setup is TTY-driven (`paxm setup`), writes agent-specific integration files (e.g. structurally merges into `~/.claude/settings.json` with a one-time `.paxm.bak` backup — `architecture.md:420-422`), and `paxm uninstall` reverses exactly that surface while leaving provider config, memory data, and telemetry untouched (`architecture.md:492-498`).

Local telemetry is privacy-conservative by default: event logs store query length and a hash prefix, not raw query text, unless `capture_query_preview: true` is explicitly set (`architecture.md:529-536`).

## What's inside this submodule

| Path | What's there |
|---|---|
| `internal/memory/types.go`, `lifecycle.go`, `ranking.go`, `router.go` | Provider contract, attribution round-tripping, ranking calibration, the router (~2.4k LOC total) |
| `internal/capturequeue/queue.go` | SQLite WAL durable event/episode/delivery queue (~1.1k LOC) |
| `internal/capture/` | Hook-facing `Runtime`, delivery + cleanup background workers, session-state bootstrap (~1.25k LOC) |
| `internal/adapters/{sqlite,zep,mem0,mem0cloud,memos,openviking,jsonrpc}/` | Provider adapters; `internal/adapters/contracttest/` is the shared black-box conformance suite every adapter runs |
| `internal/adapters/sqlite/retrieval/` | FTS5 + bounded lexical fallback + excerpt-selection (deep module, doc'd in `docs/architecture.md:98-139`) |
| `internal/mcp/server.go` | Stdio MCP server, 4 tools (~600 LOC) |
| `internal/cli/`, `cmd/paxm/` | CLI command surface grouped by audience (operator / agent tools / internal transport) |
| `internal/eval/` | Suite runner, LoCoMo integration, provider-scoped cleanup/manifests |
| `internal/config/` | YAML config model, recall/write profile validation |
| `internal/telemetry/`, `internal/dashboard/` | Bounded local logs/metrics; read-only localhost telemetry viewer |
| `docs/architecture.md`, `docs/provider-adapter-contract.md`, `docs/jsonrpc-provider-protocol.md` | The best-documented architecture doc in this study — read these before the code |
| `.claude-plugin/`, `.agents/plugins/`, `plugins/paxm-claude/`, `plugins/paxm-memory/`, `skills/paxm/` | Per-harness plugin manifests |

Total non-test Go: ~26.7k LOC across the whole module (`find ... | xargs wc -l`). Read order if you want the core: `docs/architecture.md` → `internal/memory/types.go` → `router.go` → `ranking.go` → `internal/capturequeue/queue.go` → `internal/capture/runtime.go`.

## Mental model for using it well

- **Start on SQLite.** It's the zero-dependency default and the only adapter with the deepest, most bespoke retrieval logic (FTS5 + lexical fallback + excerpt selection) — moving to a remote provider trades that away for whatever recall quality that vendor offers.
- **Use `stm` for task-local state, `ltm` for durable facts.** The config layer enforces this split structurally (expiry required on `stm`, forbidden on `ltm`) rather than leaving it to convention.
- **Don't expect semantic dedup.** If your workflow needs "these two paraphrases are the same fact," Paxm's exact-fingerprint admission won't catch it — that's Neo's or Mem0's job, not Paxm's.
- **MCP tools are intentionally underpowered relative to the CLI.** If you need setup/backfill/routing changes from inside an agent session, that's a deliberate wall, not a missing feature.
- **The queue survives crashes; recall does not retroactively fix a bad write.** `RecoverDelivering` reclaims stuck deliveries, but there's no equivalent "fix a bad memory after the fact" mechanism — Paxm doesn't reconcile facts, only redeliver them.

## When NOT to reach for this

- **You want the memory *store* to be sophisticated, not just the routing.** Reach for Mem0, Graphiti, Letta, or Neo — Paxm explicitly delegates storage sophistication to whichever provider you configure.
- **You need semantic supersession or fact reconciliation.** Paxm's admission is exact-match only; see Neo or RetainDB instead.
- **Your harness isn't one of Codex/Claude Code/OpenCode/Pi/MCP.** The value proposition is specifically the multi-harness hook integration; without that, you're just using a provider router with extra steps.
- **You want everything in one file you can `cat`.** Unlike Neo's plain-JSON fact files, Paxm's durable state lives in a SQLite WAL database (`hooks/capture.sqlite` by default) — inspectable with `sqlite3`, but not `cat`-and-`jq` simple.

## How this compares to the rest of the study

| | Paxm | Neo | Volt | Mem0 |
|---|---|---|---|---|
| **What it is** | Router + durable transport in front of pluggable stores | Reasoning engine with its own fact store | Immutable log + summary DAG (its own store) | Hybrid vector+graph+KV store |
| **Storage ownership** | None by default preference — SQLite is one adapter among 7+ | Owns it (plain JSON) | Owns it (Postgres) | Owns it (hybrid) |
| **Cross-store ranking** | Yes — the whole point (squared-reciprocal-rank calibration) | N/A, single store | N/A, single store | N/A, single store |
| **Write durability** | Local SQLite-WAL queue, async delivery, crash-recoverable | Atomic tempfile-rename | Immutable append | Direct API write |
| **Semantic reconciliation** | None (exact-fingerprint only) | Cosine-0.85 deterministic supersession | None (immutability is the point) | Adaptive updates |
| **Harness integration breadth** | Widest in the study (4 harnesses + MCP) | Claude Code + Codex plugins | Coding-agent focus, narrower | General-purpose API |

## One-line summary

> Paxm is a multi-harness hook-integration layer with a squared-reciprocal-rank cross-provider ranking calibrator and a checksummed, crash-recoverable local write queue in front of seven-plus pluggable memory backends — the study's clearest example of "memory as routing and durability," not "memory as a bespoke store."
