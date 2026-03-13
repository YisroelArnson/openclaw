# AI Personal Trainer Gateway and Memory Architecture Specification (NLSpec)

This document is a language-agnostic NLSpec for implementing the AI Personal Trainer backend architecture inspired by OpenClaw, adapted to a Postgres-first, API-plus-worker system. It is intended to be implementable from scratch by human engineers and coding agents.

---

## Table of Contents

1. [Overview and Goals](#1-overview-and-goals)
2. [System Scope and Architecture](#2-system-scope-and-architecture)
3. [Core Domains and Modules](#3-core-domains-and-modules)
4. [Data Model and Storage Contracts](#4-data-model-and-storage-contracts)
5. [Session and Transcript Lifecycle](#5-session-and-transcript-lifecycle)
6. [Queueing, Workers, and Concurrency Control](#6-queueing-workers-and-concurrency-control)
7. [Memory Model and Lifecycle](#7-memory-model-and-lifecycle)
8. [Indexing and Retrieval](#8-indexing-and-retrieval)
9. [Delivery and Streaming](#9-delivery-and-streaming)
10. [Policy, Plan Tiers, and Cost Controls](#10-policy-plan-tiers-and-cost-controls)
11. [Safety, Redaction, and Compliance Controls](#11-safety-redaction-and-compliance-controls)
12. [Provider Adapter Architecture](#12-provider-adapter-architecture)
13. [Out of Scope (Current Version)](#13-out-of-scope-current-version)
14. [Open Questions Requiring Clarification](#14-open-questions-requiring-clarification)
15. [Definition of Done](#15-definition-of-done)

---

## 1. Overview and Goals

### 1.1 Problem Statement

The AI Personal Trainer requires reliable long-term memory, robust session continuity, and auditable behavior across chat and workout interactions. A plain chat log is insufficient because the system must capture structured workout events, safety events, plan updates, and memory writes while remaining searchable and cost-controlled.

The system must support many concurrent users, preserve deterministic per-session behavior under race conditions, and provide both real-time user experience and durable final consistency. The architecture must remain implementable in Node/Express with Postgres-backed workers and optional future Redis acceleration.

### 1.2 Design Principles

**Deterministic session timelines.** Session mutation must be serialized per session and conflict-safe, with explicit locking and optimistic guards to prevent lost updates and stale parent links.

**Durable-first architecture.** Critical data must live in Postgres as source of truth. In-memory state is ephemeral and cannot be required for correctness.

**Separation of fast path and heavy path.** API ingress must remain fast and deterministic. LLM runs, indexing, compaction, and delivery retries must run in workers.

**Typed event auditability.** All significant system behavior must be stored as typed append-only events with provenance, timestamps, and actor metadata.

**Memory as layered system.** Session memory, semantic memory, daily episodic memory, and program memory must be separated but queryable through unified retrieval.

**Cost-aware by design.** Indexing and retrieval must be configurable per user plan with explicit knobs for budget and recall depth.

**Traceability over convenience.** Every retrieved chunk must trace back to session event ranges or document versions.

---

## 2. System Scope and Architecture

### 2.1 High-Level Architecture

```
+-------------------+      +---------------------+      +------------------+
| iOS/Web Clients   | ---> | API Gateway         | ---> | Postgres         |
| (chat, voice, UX) |      | (Express)           |      | (source of truth)|
+-------------------+      +---------------------+      +------------------+
            |                        |                            ^
            |                        v                            |
            |               +---------------------+               |
            |               | Redis + BullMQ      |---------------+
            |               | queues + scheduling |
            |               +---------------------+
            |                        |
            |                        v
            |               +---------------------+
            +---------------| Worker Cluster      |
                            | run/index/compact   |
                            +---------------------+
                                     |
                                     v
                            +---------------------+
                            | Redis Query Engine  |
                            | vector + FTS hybrid |
                            +---------------------+
```

### 2.2 Core Runtime Pattern

1. API receives and validates request.
2. API resolves session and appends inbound event.
3. API enqueues `agent.run_turn`.
4. Run worker performs heavy logic under per-session lock.
5. Worker appends result events and schedules follow-on jobs.
6. Streaming and final delivery inform user.

### 2.3 Transport Model

1. External product API is REST-first with SSE as the primary realtime delivery channel.
2. SSE reconnect must support resumable replay using `Last-Event-ID` header or `since` cursor.
3. Durable fetch fallback (`GET /runs/:runId`, `GET /runs/:runId/result`) is required when replay cannot be satisfied.
4. Internal orchestration uses BullMQ job contracts and typed events.
5. Optional future internal RPC control plane may be added without changing external API contracts.

### 2.3.1 API Gateway Rate Limiting and Admission Control

1. The API gateway must enforce Redis-backed token bucket rate limiting for request admission.
2. Primary throughput scope is `user_id`; secondary scopes are `device_id` and `ip_address`.
3. Authenticated requests should be primarily limited by `user_id`, with `device_id` and `ip_address` used for abuse protection and fallback enforcement.
4. The gateway must enforce concurrency admission limits separately from token bucket throughput limits.
5. Concurrency limits must include at least:
6. maximum active runs per user,
7. maximum active SSE streams per user,
8. optional maximum active SSE streams per device.
9. Rate limiting policy must be tier-adjustable through plan defaults and user overrides.
10. The gateway must return structured `429 Too Many Requests` responses with retry metadata when throughput or concurrency limits are exceeded.
11. The system must emit observability metrics for request counts, admission decisions, rate-limit hits, active runs, active streams, and tier-level usage patterns.

```pseudocode
FUNCTION admit_request(ctx: RequestContext) -> AdmissionDecision:
    identity = resolve_identity(user_id=ctx.user_id, device_id=ctx.device_id, ip_address=ctx.ip)
    policy = load_effective_policy(identity.user_id)

    IF ctx.route == "POST /messages":
        bucket_ok = token_bucket_check(
            redis,
            key="rl:user:" + identity.user_id,
            capacity=policy.rate_limit_messages_capacity,
            refill_per_second=policy.rate_limit_messages_refill_per_second,
        )
        IF NOT bucket_ok:
            RETURN reject_429(scope="user", retry_after_seconds=bucket_ok.retry_after_seconds)

    IF active_runs(identity.user_id) >= policy.concurrency_max_active_runs:
        RETURN reject_429(scope="concurrency_active_runs", retry_after_seconds=policy.retry_hint_seconds)

    IF ctx.route == "GET /runs/:runId/stream" AND
       active_streams(identity.user_id) >= policy.concurrency_max_active_streams:
        RETURN reject_429(scope="concurrency_active_streams", retry_after_seconds=policy.retry_hint_seconds)

    RETURN allow()
```

### 2.4 UI and UX Model

1. The product UI is a single-surface coach experience built as a chat feed with inline, backend-driven components.
2. The primary user interaction methods are voice, text, and lightweight taps on inline components.
3. The screen must include a persistent input dock that keeps voice and text entry available at all times.
4. The persistent input dock should support microphone state, text entry, send/submit behavior, and context-sensitive quick actions without leaving the main feed.
5. The frontend must not be organized as many separate product screens for workout logging, planning, and review; these states should appear within one continuous conversation surface.
6. The backend decides when to return plain assistant messages versus structured feed items such as exercise cards, rest timers, workout summaries, program updates, and insight cards.
7. The frontend is primarily a renderer of ordered feed items and a dispatcher of user actions back to the backend.
8. During a workout, the current exercise should be the dominant inline object in the feed while prior items recede into scrollable history.
9. Workout sessions should be generated as a session structure up front, with remaining exercises updated live as the user completes, skips, swaps, or modifies the workout.
10. The user should be able to speak naturally in shorthand relative to the active workout context, for example `done`, `add 5 lbs`, `swap this`, or `too hard`, without restating the full exercise context.
11. The design goal is to make working out feel like following a real trainer: the user sees what matters now, responds naturally, and does not manage software complexity.

---

## 3. Core Domains and Modules

### 3.1 Module Inventory

| Module | Responsibility | Primary Interface |
| --- | --- | --- |
| `api_gateway` | auth, validation, idempotency, enqueue | HTTP endpoints |
| `session_module` | `sessionKey/sessionId` lifecycle | service functions |
| `transcript_module` | append-only event storage | DB writes/reads |
| `queue_worker_module` | durable async execution | BullMQ jobs |
| `concurrency_module` | per-session serialization and conflict handling | lock/guard helpers |
| `agent_runtime_module` | context assembly + model/tool loop | worker handlers |
| `provider_adapter_module` | provider-specific request/stream/tool adaptations behind a stable runtime contract | adapter interfaces |
| `feed_contract_module` | typed message and inline-component payload contracts for the client feed | API schemas |
| `memory_module` | MEMORY/PROGRAM/DAILY docs + versions | memory services |
| `indexing_module` | extract/redact/chunk/embed/upsert | worker handlers |
| `retrieval_module` | hybrid vector + FTS search | retrieval API |
| `delivery_module` | streaming + durable final delivery | SSE + outbox |
| `policy_module` | plan-tier controls and budgets | policy resolver |
| `safety_module` | risk handling and redaction controls | policy + filters |
| `observability_module` | logs, metrics, traces, admin ops | telemetry + admin APIs |

### 3.2 Module Dependency Direction

1. API gateway may call session, transcript, policy, queue.
2. Workers may call session, transcript, memory, indexing, retrieval, delivery, safety, policy.
3. Lower-level modules must not depend on API transport details.

---

## 4. Data Model and Storage Contracts

### 4.1 Primary Tables

1. `users`
2. `user_plan_settings`
3. `session_state`
4. `session_events`
5. `runs`
6. `memory_docs`
7. `memory_doc_versions`
8. `memory_chunks`
9. `session_index_chunks`
10. `embedding_cache`
11. `retrieval_queries`
12. `retrieval_results_audit`
13. `stream_events`
14. `delivery_outbox`
15. `idempotency_keys`
16. `safety_flags`

### 4.1.1 Data Residency and Durability

1. Postgres is canonical source of truth for session state, events, memory docs, provenance, and watermarks.
2. Redis is an operational and derived layer for queue transport/scheduling, rate limiting, and cache-backed acceleration.
3. Redis data must be reconstructable from Postgres; Redis loss must not cause permanent data loss.
4. Redis cache entries and acceleration indexes must be treated as disposable derived state with Postgres fallback on cache miss.
5. Retrieval/index workers must support full Redis index rebuild from Postgres snapshots/event windows.

### 4.2 Key Table Contracts

### `session_state` (mutable current state)

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `session_key` | `String` | none | stable routing identity (`user:<id>:main`) |
| `current_session_id` | `String` | generated | active transcript instance |
| `leaf_event_id` | `String \| None` | `NONE` | current active leaf |
| `session_version` | `Integer` | `0` | optimistic guard counter |
| `updated_at` | `Timestamp` | `NOW()` | last mutation time |
| `compaction_count` | `Integer` | `0` | number of compactions |
| `memory_flush_at` | `Timestamp \| None` | `NONE` | last flush time |
| `memory_flush_compaction_count` | `Integer` | `0` | compaction count at last flush |

### `session_events` (append-only history)

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `event_id` | `String` | generated | immutable event identity |
| `session_key` | `String` | none | partition/routing key |
| `session_id` | `String` | none | transcript instance |
| `parent_event_id` | `String \| None` | `NONE` | tree link |
| `seq_num` | `Integer` | generated | monotonic sequence per session |
| `event_type` | `String` | none | typed catalog name |
| `actor` | `String` | none | user/assistant/system/tool |
| `payload` | `Dict` | `{}` | event-specific data |
| `occurred_at` | `Timestamp` | none | logical occurrence time |
| `recorded_at` | `Timestamp` | `NOW()` | persistence time |
| `idempotency_key` | `String \| None` | `NONE` | dedupe key for safe retries |

### 4.3 Pseudocode Types

```
RECORD SessionState:
    session_key                    : String
    current_session_id             : String
    leaf_event_id                  : String | None
    session_version                : Integer
    updated_at                     : Timestamp
    compaction_count               : Integer
    memory_flush_at                : Timestamp | None
    memory_flush_compaction_count  : Integer

RECORD SessionEvent:
    event_id           : String
    session_key        : String
    session_id         : String
    parent_event_id    : String | None
    seq_num            : Integer
    event_type         : String
    actor              : String
    payload            : Dict
    occurred_at        : Timestamp
    recorded_at        : Timestamp
    idempotency_key    : String | None
```

---

## 5. Session and Transcript Lifecycle

### 5.1 Session Identity Rules

1. `sessionKey` is stable per chat lane.
2. `sessionId` rotates on lifecycle boundaries.
3. `sessionKey` must be canonicalized and lowercased.
4. Alias normalization must prevent fragmented history.

### 5.2 Rotation Triggers

1. Manual reset request.
2. Idle expiry threshold.
3. Optional day-boundary policy.
4. Safety-driven forced reset (optional policy).

### 5.3 Session Resolve Algorithm

```
FUNCTION resolve_or_create_session(user_id: String, now: Timestamp) -> SessionState:
    session_key = canonicalize("user:" + user_id + ":main")
    state = SELECT session_state WHERE session_key = session_key FOR UPDATE

    IF state is NONE:
        state = create_new_session_state(session_key, now)
        RETURN state

    IF should_rotate_session(state, now):
        state.current_session_id = generate_uuid()
        state.leaf_event_id = NONE
        state.session_version = state.session_version + 1
        state.updated_at = now
        UPDATE session_state SET state
        RETURN state

    RETURN state
```

### 5.4 Transcript Append Contract

1. Events must be append-only.
2. Parent link must reference current leaf unless explicit branch operation.
3. Leaf update must be optimistic and atomic.
4. Duplicate idempotency keys must not create duplicate events.

```
FUNCTION append_session_event(state: SessionState, event: SessionEventInput) -> SessionEvent:
    TRY_LOCK_SESSION(state.session_key)
    expected_leaf = state.leaf_event_id
    new_event = insert_event_with_parent(expected_leaf, event)
    updated = UPDATE session_state
              SET leaf_event_id = new_event.event_id,
                  session_version = session_version + 1,
                  updated_at = NOW()
              WHERE session_key = state.session_key
                AND leaf_event_id IS expected_leaf
    IF updated == 0:
        RAISE ConflictError("stale leaf")
    RETURN new_event
```

### 5.5 Event Catalog Policy Flags

Every event type must define:

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `context_includable` | `Boolean` | `false` | eligible for model context assembly |
| `indexable` | `Boolean` | `false` | eligible for indexing pipeline |
| `audit_only` | `Boolean` | `false` | stored for audit only |

---

## 6. Queueing, Workers, and Concurrency Control

### 6.1 Job Model

1. API enqueues jobs into Redis-backed BullMQ queues.
2. Workers claim and process jobs asynchronously.
3. Jobs must be idempotent and retry-safe.
4. Queue runtime state is non-canonical; durable run/session outcomes must be persisted in Postgres.
5. All user actions that require an assistant response must execute through `agent.run_turn`; the system does not define a separate direct-response execution path.

### 6.2 Required Job Types

1. `agent.run_turn`
2. `memory.index_session_delta`
3. `memory.index_doc`
4. `session.compact`
5. `memory.flush_pre_compaction`
6. `memory.flush_session_end`
7. `delivery.send`
8. `delivery.retry`

### 6.3 Concurrency Guardrails

1. Per-session advisory transaction lock.
2. Optimistic leaf/version guard.
3. Idempotency key uniqueness and retry backoff.

```
FUNCTION process_session_mutating_job(job):
    lock_ok = pg_try_advisory_lock(hash(job.session_key))
    IF NOT lock_ok:
        requeue_with_backoff(job)
        RETURN

    TRY:
        execute_job(job)
    CATCH transient_error:
        retry(job)
    CATCH permanent_error:
        dead_letter(job)
    FINALLY:
        release_advisory_lock(hash(job.session_key))
```

### 6.4 Worker Modes

1. Blocking lock mode: wait for lock and continue.
2. Try-lock mode: fail fast and requeue.
3. System default should use try-lock with bounded exponential backoff.

---

## 7. Memory Model and Lifecycle

### 7.1 Memory Layers

1. `MEMORY` doc for semantic user facts/preferences.
2. `PROGRAM` doc for current training program.
3. `DAILY` docs keyed by user-local date (`YYYY-MM-DD`).
4. Session events as episodic searchable source.

### 7.2 Durable Document Contract

```
RECORD MemoryDoc:
    doc_id            : String
    user_id           : String
    doc_type          : String              -- MEMORY | PROGRAM | DAILY
    doc_key           : String              -- e.g., "DAILY:2026-02-25"
    current_version   : Integer
    updated_at        : Timestamp

RECORD MemoryDocVersion:
    doc_id            : String
    version           : Integer
    content           : String
    hash              : String
    updated_by_actor  : String
    updated_by_run_id : String | None
    created_at        : Timestamp
```

### 7.3 Daily Memory Creation Rules

1. Daily docs are created by actual writes, not empty daily pre-creation.
2. Trigger sources:
3. pre-compaction flush
4. session-end flush
5. user/manual edits
6. agent explicit memory writes

### 7.4 Read Rules for New Session Start

System should support configurable read strategy:

1. `today_only`
2. `today_and_yesterday`
3. `current_week`
4. `custom_window_days`

---

## 8. Indexing and Retrieval

### 8.1 Dirty Tracking and Sync

1. Mark changed source as dirty.
2. Coalesce via debounce.
3. Track per-session deltas (`pending_bytes`, `pending_messages`, `last_indexed_seq`).
4. Promote to index when thresholds crossed.
5. Indexing writes derived chunk/vector/search rows to Redis query indexes.
6. Postgres keeps durable indexing watermarks and provenance contracts.

### 8.2 Session Indexing Pipeline

```
FUNCTION sync_dirty_sessions():
    dirty_sessions = load_dirty_session_set()

    FOR EACH session IN dirty_sessions:
        window = select_events_after(session.last_indexed_seq)
        extracted = extract_indexable_text(window)
        redacted = redact_sensitive(extracted)
        canonical_hash = hash(redacted)

        IF canonical_hash == session.last_index_hash:
            clear_dirty(session)
            CONTINUE

        chunks = deterministic_chunk(redacted)
        embeddings = embed_with_cache(chunks, provider_model_key)
        upsert_session_index_chunks(session, chunks, embeddings)
        update_watermarks(session, window.max_seq, canonical_hash)
        clear_dirty(session)
```

### 8.3 Markdown Memory Indexing Pipeline

1. On doc version change, mark memory doc dirty.
2. Re-extract content from latest doc version.
3. Re-chunk deterministically.
4. Reuse embedding cache on unchanged chunk hashes.
5. Upsert changed chunks and delete stale chunks.

### 8.4 Retrieval Behavior

1. Hybrid search uses vector and FTS/BM25.
2. Candidate pool = `maxResults * candidateMultiplier`.
3. Merge by `chunk_id`.
4. Score = weighted vector + weighted FTS + optional recency adjustment.
5. Return top `maxResults` above `minScore`.
6. Retrieval backend mode is configurable:
7. `redis_hybrid` (default): query Redis-backed cache/index acceleration layers.
8. `postgres_fallback`: run degraded retrieval against Postgres-backed durable index tables.
9. Returned results must preserve Postgres provenance IDs/ranges even when ranked in Redis.

### 8.5 Retrieval Provenance Contract

Each result must include:

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `source_type` | `String` | none | sessions/memory/program/daily |
| `source_id` | `String` | none | session/doc identity |
| `start_seq_or_offset` | `Integer` | none | provenance start |
| `end_seq_or_offset` | `Integer` | none | provenance end |
| `chunk_id` | `String` | none | indexed chunk identity |
| `score` | `Float` | none | final rank score |

---

## 9. Delivery and Streaming

### 9.1 Dual Path Model

1. SSE stream path is primary for active UX.
2. Durable run/result fetch endpoints are mandatory fallback when SSE is interrupted or replay window expires.
3. Durable final delivery path is guaranteed via outbox.

### 9.2 Stream Events

1. `run.started`
2. `assistant.delta`
3. `tool.delta`
4. `run.completed`
5. `run.failed`
6. `heartbeat`

### 9.3 SSE Replay and Reconnect Contract

1. Every emitted stream event must include monotonic `event_id` scoped to `run_id`.
2. Stream endpoint must accept resume cursor via `Last-Event-ID` header or `since` query parameter.
3. On reconnect, server must replay missed events from `stream_events` before attaching live tail.
4. If requested cursor is older than retention window, server returns `409 replay_window_expired`.
5. Client must then fetch durable state via run/result endpoints and resume future streams from latest cursor.
6. SSE heartbeats should be emitted every `15-30s` to detect stale connections.

### 9.4 Outbox Guarantees

1. Final user-visible response must be persisted.
2. Delivery attempts must be retried with backoff.
3. Duplicate sends must be prevented by idempotency key.

```
FUNCTION finalize_run_delivery(run_id: String, final_payload: Dict):
    persist_final_run_state(run_id, final_payload)
    insert_delivery_outbox(run_id, final_payload, status="queued")
```

---

## 10. Policy, Plan Tiers, and Cost Controls

### 10.1 Effective Policy Resolution

Policy hierarchy:

1. global defaults
2. plan defaults
3. per-user overrides

### 10.2 Required Policy Keys

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `sessionIndexing.enabled` | `Boolean` | `true` | enable transcript indexing |
| `sync.sessions.deltaBytes` | `Integer` | `4096` | byte threshold |
| `sync.sessions.deltaMessages` | `Integer` | `5` | message threshold |
| `query.maxResults` | `Integer` | `8` | final retrieval result count |
| `query.candidateMultiplier` | `Integer` | `4` | candidate pool multiplier |
| `query.backend` | `String` | `"redis_hybrid"` | retrieval backend (`redis_hybrid` or `postgres_fallback`) |
| `sources` | `List<String>` | `["sessions","memory","program","daily"]` | enabled retrieval sources |
| `embedding.monthlyBudget` | `Integer` | plan-defined | optional embedding spend cap |
| `queue.retry.maxAttempts` | `Integer` | `8` | max retry attempts before dead-letter |
| `queue.retry.baseDelayMs` | `Integer` | `1000` | initial retry delay |
| `queue.retry.maxDelayMs` | `Integer` | `300000` | retry delay ceiling |
| `rateLimit.messages.capacity` | `Integer` | plan-defined | token bucket capacity for message submissions per user |
| `rateLimit.messages.refillPerSecond` | `Float` | plan-defined | token refill rate for message submissions per user |
| `rateLimit.messages.deviceCapacity` | `Integer` | plan-defined | optional device-level token bucket capacity |
| `rateLimit.messages.deviceRefillPerSecond` | `Float` | plan-defined | optional device-level token refill rate |
| `rateLimit.messages.ipCapacity` | `Integer` | plan-defined | fallback or abuse-control token bucket capacity per IP |
| `rateLimit.messages.ipRefillPerSecond` | `Float` | plan-defined | fallback or abuse-control token refill rate per IP |
| `concurrency.maxActiveRuns` | `Integer` | plan-defined | maximum concurrent active runs per user |
| `concurrency.maxActiveStreams` | `Integer` | plan-defined | maximum concurrent SSE streams per user |
| `concurrency.maxActiveStreamsPerDevice` | `Integer` | plan-defined | maximum concurrent SSE streams per device |
| `rateLimit.retryHintSeconds` | `Integer` | `30` | retry hint included in `429` responses |

### 10.3 Enforcement Rules

1. Policy checks must happen server-side.
2. Blocked features must degrade gracefully.
3. Budget enforcement must be auditable.
4. Rate limiting and concurrency admission decisions must be auditable.
5. Request admission metrics must be emitted by route, tier, and decision outcome.

---

## 11. Safety, Redaction, and Compliance Controls

### 11.1 Redaction Rules

1. Indexing must redact sensitive data classes before chunk persistence.
2. Raw events remain in audit store only under authorized access policies.
3. Retrieval must not return disallowed classes.

### 11.2 Safety Runtime Rules

1. Critical injury/emergency signals must trigger escalation behavior.
2. Safety events must be appended as `safety.*` events.
3. Unsafe recommendation paths should be interrupted or constrained by policy.

### 11.3 Safety/Redaction Audit Fields

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `safety_rule_version` | `String` | none | rule-set revision used |
| `redaction_rule_version` | `String` | none | redaction revision used |
| `decision` | `String` | none | allow/redact/block/escalate |
| `reason_codes` | `List<String>` | `[]` | deterministic justification codes |

---

## 12. Provider Adapter Architecture

### 12.1 Goal and Scope

1. Provider portability is in scope and required for the first architecture pass.
2. The implementation must support OpenAI, Anthropic, and Gemini through a stable runtime contract.
3. The core architecture (sessions, queueing, memory, indexing, retrieval, delivery) must not change when switching providers.

### 12.2 Adapter Layer Contract

The adapter layer must isolate provider differences in five areas:

1. request construction and model capability checks,
2. transcript hygiene and message ordering constraints,
3. tool schema/definition translation,
4. stream event normalization,
5. provider error classification and retryability.

### 12.3 Pseudocode Interfaces

```pseudocode
INTERFACE ProviderAdapter:
    FUNCTION provider_name() -> String
    FUNCTION validate_capabilities(request: RunRequest, caps: ProviderCapabilities) -> Void
    FUNCTION build_request(runtime_input: RuntimeInput) -> ProviderRequest
    FUNCTION normalize_stream_event(provider_event: ProviderEvent) -> NormalizedStreamEvent | None
    FUNCTION classify_error(err: Any) -> ErrorClass

INTERFACE TranscriptHygieneAdapter:
    FUNCTION apply_hygiene(messages: List<TranscriptMessage>, policy: HygienePolicy) -> List<TranscriptMessage>

INTERFACE ToolSchemaAdapter:
    FUNCTION to_provider_tools(tools: List<ToolDefinition>, caps: ProviderCapabilities) -> List<ProviderToolDefinition>

INTERFACE ErrorClassificationAdapter:
    FUNCTION classify(err: Any) -> ErrorClass

INTERFACE OutputNormalizationAdapter:
    FUNCTION normalize(provider_output: Any) -> NormalizedOutput
```

### 12.4 Provider Capability Registry

The implementation must keep a provider/model capability registry used by `agent_runtime_module` and adapters.

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `provider` | `String` | none | provider identifier (`openai`, `anthropic`, `gemini`) |
| `model` | `String` | none | provider model name |
| `supports_tools` | `Boolean` | `true` | whether native tool calling is supported |
| `supports_parallel_tools` | `Boolean` | `false` | whether concurrent tool calls are supported |
| `supports_reasoning_tokens` | `Boolean` | `false` | whether provider exposes reasoning token controls |
| `max_context_tokens` | `Integer` | none | hard context limit used in runtime checks |
| `stream_protocol` | `String` | none | provider streaming shape classification |

### 12.5 Runtime Flow with Adapters

```pseudocode
FUNCTION run_agent_turn(run_input: RuntimeInput) -> RunResult:
    adapter = provider_adapter_module.get_adapter(run_input.provider)
    caps = provider_adapter_module.get_capabilities(run_input.provider, run_input.model)

    adapter.validate_capabilities(run_input.request, caps)
    hydrated_messages = transcript_hygiene_adapter.apply_hygiene(run_input.messages, run_input.hygiene_policy)
    provider_tools = tool_schema_adapter.to_provider_tools(run_input.tools, caps)
    provider_request = adapter.build_request(RuntimeInput(messages=hydrated_messages, tools=provider_tools, ...))

    TRY:
        provider_stream = provider_client.execute(provider_request)
    CATCH err:
        error_class = error_classification_adapter.classify(err)
        RETURN RunResult(status="error", error_class=error_class)

    FOR EACH provider_event IN provider_stream:
        norm = adapter.normalize_stream_event(provider_event)
        IF norm is not NONE:
            delivery_module.emit_stream_event(norm)

    RETURN output_normalization_adapter.normalize(provider_stream.final_output())
```

### 12.6 Non-Negotiable Invariants

1. Switching provider must not require schema changes in session, memory, or indexing tables.
2. Switching provider must preserve transcript append contracts and session concurrency behavior.
3. Provider-specific features may be enabled via capability flags, but unsupported features must degrade gracefully.
4. Errors must map to stable internal error classes used by retry/backoff policies.

---

## 13. Out of Scope (Current Version)

**Real-time collaborative multi-user program editing.** Concurrent multi-author real-time editing of `PROGRAM` is out of scope. Extension point: OT/CRDT collaboration layer on top of `memory_doc_versions`.

**Provider-specific product UX surfaces.** Provider-branded advanced UX controls are out of scope for first pass. Extension point: provider-specific admin/settings UI.

**Cross-tenant federated memory sharing.** Shared memory graphs across users/coaches are out of scope. Extension point: separate `shared_memory` domain with explicit ACLs.

**Automated medical diagnosis workflows.** Medical diagnostic automation is out of scope. Extension point: regulated safety escalation integration and clinician review workflow.

---

## 14. Open Questions Requiring Clarification

1. Should `sessionId` day-boundary rotation be enabled by default, or only idle/manual reset for v1 rollout?
2. For daily memory reads at new-session start, should default be `today_and_yesterday` or `current_week`?
3. Should users be notified every time a pre-compaction/session-end memory flush occurs, or only when user-visible summary is generated?
4. What retention policy should apply to raw `session_events` and indexed chunks by plan tier?
5. Do you want branch navigation exposed in user-facing UX now, or keep tree structure internal only?
6. Should `PROGRAM` edits require user confirmation gates for specific risky changes (e.g., large load jumps)?
7. What exact monthly embedding budget defaults do you want for each plan tier?
8. Should try-lock requeue backoff use fixed intervals or exponential with jitter as mandatory behavior?

---

## 15. Definition of Done

### 15.1 API Gateway and Session Resolve

- [ ]  API validates auth, schema, and idempotency before mutation.
- [ ]  API resolves canonical `sessionKey` and active `sessionId` for each inbound action.
- [ ]  API persists inbound event and enqueues worker jobs in one request flow.
- [ ]  API returns deterministic `202 Accepted` with `sessionKey`, `runId`, `jobId`, and stream URL.
- [ ]  API gateway enforces Redis-backed token bucket rate limits for message submissions.
- [ ]  API gateway enforces active-run and active-stream concurrency limits per effective user policy.
- [ ]  `429` responses include structured retry metadata and limit scope.

### 15.2 Transcript and Event Model

- [ ]  `session_events` is append-only and immutable after insert.
- [ ]  Parent linkage is enforced and leaf pointer advances atomically.
- [ ]  Event catalog supports `context_includable`, `indexable`, and `audit_only` flags.
- [ ]  Workout, program, memory, safety, and system event families are implemented.

### 15.3 Queue, Workers, and Concurrency

- [ ]  BullMQ jobs in Redis are durable enough for restart/recovery under configured persistence/HA policy.
- [ ]  Per-session advisory locks prevent concurrent mutation of same session.
- [ ]  Optimistic leaf/version guard rejects stale writes.
- [ ]  Idempotency and bounded retries prevent duplicate side effects.
- [ ]  Failed jobs are observable and routed to dead-letter handling.
- [ ]  Queue state is recoverable without data loss because canonical outcomes remain in Postgres.

### 15.4 Memory and Document Lifecycle

- [ ]  `MEMORY`, `PROGRAM`, and `DAILY` docs exist as versioned DB records.
- [ ]  Pre-compaction and session-end flush paths can create/update daily docs.
- [ ]  New-session memory reads follow configured window policy.
- [ ]  Memory writes are attributable to actor and run/session provenance.

### 15.5 Indexing and Retrieval

- [ ]  Dirty tracking with delta thresholds gates indexing work.
- [ ]  Session indexing extracts, redacts, chunks, embeds, and upserts with provenance.
- [ ]  Memory doc indexing reuses embedding cache via provider/model/config+chunk hash.
- [ ]  Redis hybrid retrieval (vector + FTS/BM25) returns ranked results with Postgres provenance.
- [ ]  Retrieval can be stale only within async sync window, with visible watermark metrics.
- [ ]  Full Redis index rebuild from Postgres is supported and documented.

### 15.6 Delivery and Streaming

- [ ]  Streaming emits lifecycle and delta events during run execution.
- [ ]  SSE reconnect supports resume via `Last-Event-ID` or `since` cursor.
- [ ]  Missed events are replayed from durable `stream_events` before live tail.
- [ ]  Replay-window expiry returns deterministic `409 replay_window_expired`.
- [ ]  Final run output is durably persisted regardless of stream disconnect.
- [ ]  Outbox retries transient failures and records terminal failures.
- [ ]  Duplicate final deliveries are prevented by idempotency constraints.

### 15.7 Policy, Cost, and Safety Enforcement

- [ ]  Effective policy resolution merges global, plan, and user overrides.
- [ ]  Plan-tier knobs control indexing, retrieval, sources, and embedding budget.
- [ ]  Plan-tier knobs control request rate limits and concurrency admission limits.
- [ ]  Redaction rules execute before indexed storage.
- [ ]  Safety signals trigger deterministic escalation/constraint behavior.
- [ ]  All policy, safety, and redaction decisions are auditable.

### 15.8 Provider Adapter Portability

- [ ]  Provider adapter layer supports OpenAI, Anthropic, and Gemini via one stable runtime contract.
- [ ]  Transcript hygiene is provider-aware and deterministic before each model run.
- [ ]  Tool definitions are transformed per provider from one canonical internal tool schema.
- [ ]  Stream events are normalized to one internal event model used by delivery.
- [ ]  Provider errors map to stable retryable/non-retryable/internal classes.
- [ ]  Switching providers requires no changes to session, memory, indexing, or retrieval schemas.

### 15.9 Cross-Feature Parity Matrix

| Test Case | OpenAI | Anthropic | Gemini | Memory-Only Plan | Premium Hybrid Plan |
| --- | --- | --- | --- | --- | --- |
| Inbound user message accepted and persisted | [ ] | [ ] | [ ] | [ ] | [ ] |
| Per-session lock prevents concurrent head corruption | [ ] | [ ] | [ ] | [ ] | [ ] |
| Daily memory flush on compaction cycle | [ ] | [ ] | [ ] | [ ] | [ ] |
| Session indexing obeys delta thresholds | [ ] | [ ] | [ ] | [ ] | [ ] |
| Hybrid retrieval returns traceable provenance | [ ] | [ ] | [ ] | [ ] | [ ] |
| Final delivery survives stream disconnect | [ ] | [ ] | [ ] | [ ] | [ ] |
| SSE reconnect replays missed events and resumes live | [ ] | [ ] | [ ] | [ ] | [ ] |
| Rate limiting returns structured `429` with retry metadata | [ ] | [ ] | [ ] | [ ] | [ ] |
| Budget limit blocks embedding with graceful degradation | [ ] | [ ] | [ ] | [ ] | [ ] |
| Safety escalation path emits safety events | [ ] | [ ] | [ ] | [ ] | [ ] |
| Canonical tool definitions execute through adapter mappings | [ ] | [ ] | [ ] | [ ] | [ ] |

### 15.10 Integration Smoke Test

```
-- Setup
user_id = "u_123"
policy = load_effective_policy(user_id)
ASSERT policy is not NONE

-- Inbound action
resp = POST /messages { user_id, text="Completed squat set 225x5 RPE 8", idempotency_key="k1" }
ASSERT resp.status == 202
ASSERT resp.session_key == "user:u_123:main"
ASSERT resp.job_id is not NONE
ASSERT resp.stream_url is not NONE

-- Rate limiting metadata
limited = POST /messages { user_id, text="spam", idempotency_key="k-limit" } after exhausting bucket
ASSERT limited.status == 429
ASSERT limited.error.scope is not NONE
ASSERT limited.error.retry_after_seconds > 0

-- Worker run and lock behavior
job = dequeue("agent.run_turn")
ASSERT job.session_key == "user:u_123:main"
lock_ok = pg_try_advisory_lock(hash(job.session_key))
ASSERT lock_ok == true

-- Transcript append
event = append_session_event(...)
ASSERT event.event_type == "conversation.user_message" OR event.event_type == "workout.set.completed"

-- Flush + memory write
flush_job = enqueue("memory.flush_session_end", { session_key: job.session_key })
process(flush_job)
daily_doc = get_memory_doc(user_id, "DAILY", current_user_date(user_id))
ASSERT daily_doc is not NONE

-- Indexing
index_job = enqueue("memory.index_session_delta", { session_key: job.session_key })
process(index_job)
results = retrieval_search(user_id, "squat 225x5")
ASSERT LENGTH(results) > 0
ASSERT results[0].source_id is not NONE
ASSERT results[0].chunk_id is not NONE

-- Delivery guarantee
complete_run(job.run_id)
final = fetch_run_final(job.run_id)
ASSERT final.status == "completed"
ASSERT outbox_record_exists(job.run_id)

-- SSE reconnect and replay fallback
disconnect_sse(job.run_id)
reconnect = GET /runs/{job.run_id}/stream?since=last_seen_event_id
ASSERT reconnect.status == 200
ASSERT stream_replayed_events(reconnect) == true

expired = GET /runs/{job.run_id}/stream?since=very_old_event_id
ASSERT expired.status == 409
ASSERT expired.error_code == "replay_window_expired"

fallback = GET /runs/{job.run_id}/result
ASSERT fallback.status == 200
ASSERT fallback.payload is not NONE

-- Concurrency conflict safety
start_worker_a(job.session_key)
start_worker_b(job.session_key)
ASSERT NOT (worker_a.mutates_head AND worker_b.mutates_head at same time)
```

---

## Appendix A: Event Catalog Baseline

| Event Family | Example Type | context_includable | indexable | audit_only |
| --- | --- | --- | --- | --- |
| `conversation.*` | `conversation.user_message` | `true` | `true` | `false` |
| `conversation.*` | `conversation.assistant_message` | `true` | `true` | `false` |
| `tool.*` | `tool.call` | `false` | `false` | `true` |
| `tool.*` | `tool.result` | `true` | `optional` | `false` |
| `workout.*` | `workout.set.completed` | `true` | `true` | `false` |
| `workout.*` | `workout.exercise.skipped` | `true` | `true` | `false` |
| `program.*` | `program.updated` | `true` | `true` | `false` |
| `memory.*` | `memory.flush.executed` | `false` | `false` | `true` |
| `safety.*` | `safety.alert` | `true` | `optional` | `false` |
| `system.*` | `system.retry` | `false` | `false` | `true` |
| `compaction.*` | `compaction.summary` | `true` | `true` | `false` |

---

## Appendix B: Design Decision Rationale

**Why Redis/BullMQ jobs instead of in-process only queues?** The PT app is multi-user SaaS oriented and requires durable retries, horizontal worker scaling, and restart-safe job orchestration.

**Why keep `sessionKey` with only one current user chat?** It preserves architectural extensibility and stable routing identity while allowing `sessionId` rotation without losing continuity.

**Why append-only events plus mutable session state?** This separates immutable audit history from operational mutable pointers (`leaf_event_id`, counters), improving traceability and conflict handling.

**Why SSE-first delivery with replay + outbox?** SSE gives responsive UX; replay via cursor handles mobile disconnects; outbox guarantees final consistency independent of connection state.

**Why dirty marking plus sync jobs for indexing?** It coalesces burst writes, controls embedding costs, and still guarantees eventual indexing via durable worker processing.
