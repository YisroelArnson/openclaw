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
11. [Redaction and Risk Controls](#11-redaction-and-risk-controls)
12. [Prompt Layers and Lifecycle](#12-prompt-layers-and-lifecycle)
13. [Provider Adapter Architecture](#13-provider-adapter-architecture)
14. [Out of Scope (Current Version)](#14-out-of-scope-current-version)
15. [Current Defaults and Deferred Design Areas](#15-current-defaults-and-deferred-design-areas)
16. [Definition of Done](#16-definition-of-done)

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

**Memory as layered system.** Session memory, semantic memory, date-keyed episodic memory, and program memory must be separated but queryable through unified retrieval.

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
12. The gateway must validate request schema, authentication, authorization context, required headers, idempotency metadata, and cursor/query parameter shape before queueing work.
13. Malformed or incomplete requests must be rejected at the gateway with deterministic `4xx` responses and must not reach worker queues.
14. Invalid component actions, invalid session/run ownership, and malformed SSE resume cursors must be rejected before any downstream mutation.

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

1. The product UI is a single-surface coach experience built as a chat feed with backend-driven UI elements and a persistent workout tray.
2. The primary user interaction methods are voice, text, and lightweight taps on feed items or tray actions.
3. The screen must include a persistent input dock that keeps voice and text entry available at all times.
4. The persistent input dock should support microphone state, live transcript capture, text entry, send/submit behavior, and context-sensitive quick actions without leaving the main feed.
5. The persistent input dock should include a compact quick-action affordance, a text field that can expand when the user types, and a microphone control that supports clear on/off interaction states.
6. Live speech transcription should appear in the dock text field while the user is speaking so that voice and text feel like one input system rather than separate modes.
7. The frontend must not be organized as many separate product screens for workout logging, planning, and review; these states should appear within one continuous conversation surface.
8. The backend decides when to return plain assistant messages versus structured feed items such as workout previews, workout summaries, program updates, insight cards, or other non-current-task UI components.
9. The frontend is primarily a renderer of ordered feed items plus a persistent workout tray, and a dispatcher of user actions back to the backend.
10. During a workout, the current exercise and rest state should live in the persistent workout tray rather than as the primary actionable object inline in the feed.
11. Inline feed UI should be used for contextual or reflective information, such as workout previews, summaries, insights, or program updates, not as the main control surface for the current live exercise.
12. Workout sessions should be generated as a session structure up front, with remaining exercises updated live as the user completes, skips, swaps, or modifies the workout.
13. The user should be able to speak naturally in shorthand relative to the active workout context, for example `done`, `add 5 lbs`, `swap this`, or `too hard`, without restating the full exercise context.
14. The design goal is to make working out feel like following a real trainer: the user sees what matters now, responds naturally, and does not manage software complexity.

In plain product terms, the application should look like a simple chat screen with a persistent workout tray near the bottom. Most of the screen is the conversation feed, where the trainer talks naturally and prior messages recede upward into history. During a workout, the current exercise or rest state remains persistently accessible in a compact tray above the input dock. That tray shows the current workout state, a short progress line, and one obvious action such as `complete`, `start`, or `continue`.

When the user taps the tray, the tray itself should expand upward in place and fill with more detail and actions. It is not a separate modal or bottom sheet that covers the voice/text dock. The input dock must remain visible and anchored below the tray while the tray grows upward. In its expanded state, the tray reveals more information about the current exercise, such as target reps or weight, a short coaching cue, and a small set of additional actions like `skip`, `swap`, or `make it easier`.

The overall effect should feel like one calm coaching surface: the chat handles the relationship and guidance, inline feed components handle contextual information, the persistent workout tray handles the current live workout action, and the input dock remains continuously available for natural voice or text interaction.

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
| `bff_module` | aggregate frontend-ready view models and feed payloads so clients do not orchestrate many backend calls | API view-model services |
| `memory_module` | MEMORY/PROGRAM/date-keyed episodic docs + versions | memory services |
| `trainer_tools_module` | typed trainer action catalog and execution handlers | tool registry + handlers |
| `workout_module` | live workout state, exercise progression, and tray-backed current action state | workout services |
| `indexing_module` | extract/redact/chunk/embed/upsert | worker handlers |
| `retrieval_module` | hybrid vector + FTS search | retrieval API |
| `delivery_module` | streaming + durable final delivery | SSE + outbox |
| `policy_module` | plan-tier controls and budgets | policy resolver |
| `observability_module` | logs, metrics, traces, admin ops | telemetry + admin APIs |

### 3.2 Module Dependency Direction

1. API gateway may call session, transcript, policy, queue.
2. API gateway and BFF module may call session, transcript, memory, retrieval, delivery, and policy to assemble client-ready payloads.
3. Workers may call session, transcript, memory, indexing, retrieval, delivery, safety, policy.
4. Lower-level modules must not depend on API transport details.

### 3.3 Backend-for-Frontend Aggregation Contract

1. The backend must provide frontend-ready aggregated payloads so the client does not need to orchestrate many independent requests to render the primary coach surface.
2. BFF aggregation may compose session state, active run state, feed items, current workout, current exercise, timers, and lightweight memory/profile summaries into a single view model.
3. Aggregated client payloads must be derived from canonical Postgres state, with Redis used only as an acceleration layer for cacheable reads.
4. BFF aggregation must not create alternate business truth; it is a read-model layer over canonical state.
5. The client feed contract should prefer one ordered response payload over many chat-adjacent support requests.
6. The exact coach-surface payload schema, tray payload schema, and feed item schemas are intentionally deferred and must be defined before frontend implementation begins.

```pseudocode
FUNCTION build_coach_surface(user_id: String, session_key: String) -> CoachSurfaceView:
    session_state = load_session_state(session_key)
    active_run = load_active_run(user_id, session_key)
    feed_items = load_recent_feed_items(session_key)
    workout_state = load_current_workout_state(user_id, session_key)
    quick_stats = load_quick_stats_summary(user_id)

    RETURN CoachSurfaceView(
        session=session_state,
        run=active_run,
        feed=feed_items,
        workout=workout_state,
        quick_stats=quick_stats,
    )
```

### 3.4 Agentic Semantic Error Handling and Recovery

1. Tool execution and domain actions must handle semantic mistakes explicitly, not only transport or type errors.
2. When a tool call is syntactically valid but semantically wrong for the current workout, program, safety state, or memory state, the runtime must return a structured semantic error result.
3. Structured semantic error results must include:
4. stable machine-readable error code,
5. human-readable explanation,
6. actionable corrective guidance for the agent,
7. optional suggested constraints, alternatives, or valid next steps.
8. The runtime should pass semantic error results back into the model loop so the agent can recover and continue rather than collapsing the run immediately.
9. Semantic error events must be logged and auditable so prompts, tool contracts, and guardrails can be improved over time.
10. Repeated semantic failures within one run may trigger escalation or safe fallback according to policy.

```pseudocode
RECORD SemanticToolError:
    code                : String
    explanation         : String
    agent_guidance      : String
    suggested_fix       : Dict | None
    retryable_in_run    : Boolean

FUNCTION execute_tool_with_recovery(call: ToolCall) -> ToolResult:
    validation = validate_tool_call(call)
    IF validation.status == "semantic_error":
        error = SemanticToolError(
            code=validation.code,
            explanation=validation.explanation,
            agent_guidance=validation.agent_guidance,
            suggested_fix=validation.suggested_fix,
            retryable_in_run=true,
        )
        append_semantic_error_event(call, error)
        RETURN ToolResult(status="semantic_error", error=error)

    RETURN execute_tool(call)
```

### 3.5 Trainer Tool Registry

The trainer must act through an explicit typed tool catalog rather than vague free-form "do something" behavior.

#### 3.5.1 Tool Categories

The core tool registry should include these categories:

1. context and memory recall,
2. workout execution,
3. live workout adjustment,
4. program management,
5. coaching calculations and decision support,
6. memory and episodic-note writes.

#### 3.5.2 Core Tool Definitions

| Tool | Category | Mutating | Purpose |
| --- | --- | --- | --- |
| `memory_search` | context | `false` | semantically search durable memory and episodic notes |
| `memory_get` | context | `false` | fetch current `memory_markdown` content |
| `program_get` | context | `false` | load current `program_markdown` |
| `get_recent_workout_history` | context | `false` | load recent workout performance/history |
| `get_user_readiness` | context | `false` | load readiness/recovery signals |
| `workout_generate` | workout execution | `true` | generate a workout session structure when the user is ready to train |
| `workout_get_current_state` | workout execution | `false` | load the current live workout/tray state |
| `workout_start_session` | workout execution | `true` | mark the workout session active |
| `workout_complete_set` | workout execution | `true` | complete the current set and update progression state |
| `workout_complete_exercise` | workout execution | `true` | mark current exercise complete |
| `workout_skip_exercise` | workout execution | `true` | skip current exercise |
| `workout_start_rest` | workout execution | `true` | begin rest state for the active exercise |
| `workout_end_rest` | workout execution | `true` | end rest state and advance flow |
| `workout_finish_session` | workout execution | `true` | mark session finished and emit summary events |
| `workout_adjust_load` | live adjustment | `true` | change current or remaining load targets |
| `workout_adjust_reps` | live adjustment | `true` | change current or remaining rep targets |
| `workout_adjust_duration` | live adjustment | `true` | change current or remaining duration targets |
| `workout_swap_exercise` | live adjustment | `true` | replace the current exercise with a valid substitute |
| `workout_mark_too_hard` | live adjustment | `true` | record difficulty signal and trigger adaptation |
| `workout_mark_pain_flag` | live adjustment | `true` | record safety/pain signal and constrain future actions |
| `document_replace_text` | document mutation | `true` | replace a specific text span inside `memory_markdown` or `program_markdown` |
| `document_replace_entire` | document mutation | `true` | replace the full contents of `memory_markdown` or `program_markdown` |
| `program_adjust_progression` | program | `true` | progress/regress future training prescription |
| `calculate_estimated_1rm` | decision support | `false` | compute estimated 1RM from performance data |
| `suggest_exercise_substitution` | decision support | `false` | find context-appropriate replacement movements |
| `evaluate_readiness_adjustment` | decision support | `false` | suggest intensity or volume modifications |
| `episodic_note_append` | memory write | `true` | append a date-keyed episodic note |

#### 3.5.3 Tool Execution Rules

1. The model must choose from canonical internal tools, not invent tool names.
2. Read-only tools should be preferred before mutating tools when additional context is needed.
3. Mutating tools must validate workout/program/session state before applying changes.
4. Every mutating tool call must append auditable `tool.*` and domain events.
5. Tool outputs must be shaped so the agent can continue the run without guessing hidden side effects.

#### 3.5.3.1 DB-Backed Markdown Document Mutation Tools

The system should treat user-facing memory and program documents as versioned Markdown stored in Postgres, not as workspace files.

The canonical document-mutation tools are:

| Tool | Required Inputs | Behavior |
| --- | --- | --- |
| `document_replace_text` | `doc_key`, `old_text`, `new_text`, `expected_version`, `reason` | perform a targeted text replacement against the current Markdown version and write a new version on success |
| `document_replace_entire` | `doc_key`, `markdown`, `expected_version`, `reason` | replace the entire Markdown document and write a new version |
| `episodic_note_append` | `date_key`, `markdown_block`, `reason` | append a Markdown block to the date-keyed episodic note for that date |

These tools should follow these rules:

1. `memory_get` and `program_get` are the read path for durable Markdown documents.
2. `document_replace_text` is the preferred targeted-edit path for `memory_markdown` and `program_markdown`.
3. `document_replace_entire` is the preferred full-rewrite path for `memory_markdown` and `program_markdown`.
4. `episodic_note_append` is append-only and should be used for flushes, notes, and time-based episodic writes.
5. All document mutation tools must enforce optimistic concurrency through `expected_version`.
6. All document mutation tools must create new document versions rather than mutating rows in place.
7. The system must not expose patch-style document editing as part of the core document tool contract.

#### 3.5.4 Live Workout Generation

1. The workout session should be generated when the user indicates they are ready to work out.
2. Natural-language triggers such as "I'm ready", "start workout", or similar tray actions may lead the agent to call `workout_generate`.
3. `workout_generate` should accept guidance context such as readiness, time available, pain constraints, equipment, and current program state.
4. `workout_generate` returns a workout session structure that becomes the source of truth for the persistent workout tray.
5. After generation, the agent should be notified of the generated structure so it can coach the user through the workout conversationally.

```pseudocode
FUNCTION handle_user_ready_to_work_out(user_id: String, guidance: Dict) -> WorkoutSessionState:
    current_program = program_get(user_id)
    readiness = get_user_readiness(user_id)
    workout = workout_generate(
        user_id=user_id,
        program=current_program,
        readiness=readiness,
        guidance=guidance,
    )
    notify_agent_workout_generated(user_id, workout)
    RETURN workout
```

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
16. `risk_flags`

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

RECORD ExerciseDefinition:
    exercise_id            : String
    slug                   : String
    name                   : String
    category               : String
    movement_pattern       : String
    primary_muscles        : List<String>
    secondary_muscles      : List<String>
    equipment_required     : List<String>
    equipment_optional     : List<String>
    difficulty             : String
    unilateral             : Boolean
    default_tracking_mode  : String
    contraindications      : List<String>
    coaching_cues          : List<String>
    regression_ids         : List<String>
    progression_ids        : List<String>
    substitution_tags      : List<String>

RECORD ProgramSetTarget:
    set_index              : Integer
    target_reps            : Integer | None
    rep_range_min          : Integer | None
    rep_range_max          : Integer | None
    target_load            : Float | None
    target_duration_sec    : Integer | None
    target_distance_m      : Integer | None
    target_rpe             : Float | None

RECORD ProgramExercise:
    program_exercise_id    : String
    exercise_id            : String
    block_id               : String
    day_key                : String
    order_index            : Integer
    prescription_type      : String
    sets                   : List<ProgramSetTarget>
    rest_seconds           : Integer | None
    tempo                  : String | None
    intensity_type         : String | None
    notes                  : String | None
    progression_rule       : Dict | None

RECORD WorkoutSetState:
    set_index              : Integer
    status                 : String
    prescribed_reps        : Integer | None
    prescribed_load        : Float | None
    prescribed_duration_sec: Integer | None
    prescribed_distance_m  : Integer | None
    prescribed_rpe         : Float | None
    actual_reps            : Integer | None
    actual_load            : Float | None
    actual_duration_sec    : Integer | None
    actual_distance_m      : Integer | None
    actual_rpe             : Float | None
    started_at             : Timestamp | None
    completed_at           : Timestamp | None
    notes                  : String | None

RECORD WorkoutAdjustment:
    adjustment_id          : String
    type                   : String
    source                 : String
    reason                 : String | None
    before                 : Dict | None
    after                  : Dict | None
    created_at             : Timestamp

RECORD WorkoutExerciseState:
    workout_exercise_id    : String
    workout_session_id     : String
    program_exercise_id    : String | None
    exercise_id            : String
    status                 : String
    order_index            : Integer
    started_at             : Timestamp | None
    completed_at           : Timestamp | None
    current_set_index      : Integer | None
    prescribed             : Dict
    sets                   : List<WorkoutSetState>
    active_rest            : Dict | None
    adjustments            : List<WorkoutAdjustment>
    coach_message          : String | None

RECORD WorkoutSessionState:
    workout_session_id     : String
    user_id                : String
    status                 : String
    started_at             : Timestamp
    completed_at           : Timestamp | None
    current_exercise_index : Integer | None
    current_exercise_id    : String | None
    current_phase          : String
    exercises              : List<WorkoutExerciseState>
    summary                : Dict
```

### 4.4 Workout and Exercise State Model

The system must separate:

1. exercise library definitions,
2. program prescriptions,
3. live workout execution state.

#### 4.4.1 Exercise Definition Layer

`ExerciseDefinition` is the canonical movement-library entry. It should describe what the movement is, how it is tracked, what equipment it requires, and what substitutions or regressions are valid.

This layer should be stable reference data and should not be mutated by live workout execution.

#### 4.4.2 Program Prescription Layer

`ProgramExercise` stores how a movement is prescribed for the user within the current program.

This layer represents plan intent:

1. order in the session,
2. sets/reps/load/duration targets,
3. rest/tempo/intensity guidance,
4. progression rules.

This layer may evolve as the agent updates `program_markdown`.

The program source of truth is `program_markdown`, stored as versioned Markdown content in Postgres. The exact internal Markdown structure for the program is intentionally deferred and must be defined before implementation of program parsing and mutation logic.

#### 4.4.3 Live Workout Execution Layer

`WorkoutSessionState` and `WorkoutExerciseState` represent what is happening right now in the active workout.

This layer is what the persistent workout tray should read from and what live workout tools should modify.

It must support:

1. current exercise pointer,
2. current phase (`exercise`, `rest`, `transition`, `finished`),
3. performed set state,
4. live adjustments,
5. coach cues for the tray.

#### 4.4.4 Tool Interaction Contract

1. `workout_adjust_*` tools should mutate live workout state, not the exercise library.
2. program-related tools should mutate `program_markdown` and future training prescription, not rewrite completed workout history.
3. `workout_complete_set` and related tools should update `WorkoutSetState` and advance live workout flow.
4. The workout tray should be backed by `WorkoutSessionState.current_phase`, the current `WorkoutExerciseState`, and the current `WorkoutSetState`.

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
3. Day-boundary rotation should be enabled by default.
4. Session reset policy must be configurable per user/plan.

Default reset posture:

1. `day_boundary_enabled = true`
2. `idle_expiry_minutes = 180`
3. whichever reset boundary is reached first should trigger the next `sessionId` rotation.

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
5. Tree-linked transcript structure is internal only and must not be exposed as branch navigation in the user-facing UX.

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
4. Retry backoff must use exponential backoff with jitter.

### 6.5 Deferred Queue and Worker Design Details

The current architecture intentionally leaves some worker-system details to be defined during implementation design. These items must be specified before production implementation:

1. queue topology and queue names,
2. worker-role to queue assignment,
3. job payload envelopes and per-job schemas,
4. dead-letter and replay policy,
5. stalled-job recovery behavior,
6. durable reconciliation procedure after partial queue/runtime failure.

---

## 7. Memory Model and Lifecycle

### 7.1 Memory Layers

1. `MEMORY` doc for semantic user facts/preferences.
2. `PROGRAM` doc for current training program.
3. `EPISODIC_DATE` docs keyed by user-local date (`YYYY-MM-DD`).
4. Session events as episodic searchable source.

### 7.2 Durable Document Contract

```
RECORD MemoryDoc:
    doc_id            : String
    user_id           : String
    doc_type          : String              -- MEMORY | PROGRAM | EPISODIC_DATE
    doc_key           : String              -- e.g., "EPISODIC_DATE:2026-02-25"
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

### 7.3 Date-Keyed Episodic Note Creation Rules

1. Date-keyed episodic notes are created by actual writes, not empty daily pre-creation.
2. Trigger sources:
3. pre-compaction flush
4. session-end flush
5. user/manual edits
6. agent explicit memory writes
7. not every calendar day should have an episodic note.
8. Pre-compaction and session-end flushes should run silently in the background and should not notify the user by default.

### 7.4 Read Rules for New Session Start

System should support configurable read strategy for date-keyed episodic notes:

1. `today_only`
2. `today_and_yesterday`
3. `current_week`
4. `custom_window_days`

Default posture:

1. `today_and_yesterday` should be the default read window.
2. Read window must remain configurable per user/plan so higher tiers can use larger context windows.

### 7.5 Retention Rules

1. Raw `session_events` must be retained indefinitely by default.
2. Versioned memory and program documents must be retained indefinitely by default.
3. Indexed chunks and retrieval provenance must be retained indefinitely by default.

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
| `source_type` | `String` | none | sessions/memory/program/episodic_date |
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

### 9.5 Heartbeat and Proactive Coaching

Heartbeat is the proactive coach-awareness loop. It is distinct from cron-style scheduled jobs and is intended to decide whether the coach should surface something meaningful to the user.

#### 9.5.1 Heartbeat Purpose

1. Heartbeat should run as a periodic per-user proactive coach turn in the user's main trainer relationship context.
2. Heartbeat should be used for awareness, follow-up, gentle nudges, and timely coach messages.
3. Heartbeat should not be used for precise scheduling, heavy background analysis, or exact-time reminders.
4. The implementation should prefer silence when no meaningful proactive message is warranted.

#### 9.5.2 Pre-Run Gating

The system must apply deterministic gating before making a heartbeat LLM call.

Pre-run gating should consider at least:

1. whether the user has heartbeat/proactive coaching enabled,
2. active hours / quiet hours,
3. recent user interaction recency,
4. whether the user is in an active workout or active run,
5. recent proactive message cooldown windows,
6. whether any candidate reason exists to speak.

If no candidate reason exists, or if policy says the coach should remain quiet, the heartbeat run should be skipped before calling the model.

The exact candidate-reason taxonomy is intentionally deferred and must be defined before production implementation of heartbeat behavior.

#### 9.5.3 Heartbeat Result Contract

Heartbeat runs must return one of these logical outcomes:

1. `HEARTBEAT_OK`
2. `THREAD_ONLY`
3. `THREAD_PLUS_PUSH`

Meaning:

1. `HEARTBEAT_OK` means nothing important or timely should be surfaced to the user.
2. `THREAD_ONLY` means a proactive coach message should be appended to the user's thread, but no push notification should be sent.
3. `THREAD_PLUS_PUSH` means a proactive coach message should be appended to the user's thread and is recommended for push delivery.

The prompt should strongly bias toward `HEARTBEAT_OK` unless there is a meaningful reason to speak.

#### 9.5.4 Server-Controlled Delivery Enforcement

1. The model may recommend `THREAD_ONLY` or `THREAD_PLUS_PUSH`, but the server remains the final authority on delivery behavior.
2. The server may downgrade `THREAD_PLUS_PUSH` to `THREAD_ONLY` based on:
3. quiet hours,
4. user push-notification settings,
5. recent notification fatigue/cooldowns,
6. active-session recency,
7. other safety or policy rules.
8. If push is disabled for the user, the system may still run heartbeat and append a message to the thread when appropriate, but it must not send a push notification.
9. If the entire proactive feature is disabled for the user, the system must skip heartbeat runs entirely.

```pseudocode
FUNCTION should_run_heartbeat(user_id: String, now: Timestamp) -> Boolean:
    prefs = load_proactive_preferences(user_id)
    IF NOT prefs.heartbeat_enabled:
        RETURN false
    IF is_quiet_hours(user_id, now):
        RETURN false
    IF has_active_run(user_id) OR has_active_workout(user_id):
        RETURN false
    IF recently_interacted(user_id):
        RETURN false
    IF NOT has_candidate_heartbeat_reason(user_id, now):
        RETURN false
    RETURN true

FUNCTION enforce_heartbeat_delivery(user_id: String, recommendation: String) -> String:
    prefs = load_proactive_preferences(user_id)
    IF recommendation == "THREAD_PLUS_PUSH":
        IF NOT prefs.push_enabled:
            RETURN "THREAD_ONLY"
        IF is_quiet_hours(user_id, NOW()):
            RETURN "THREAD_ONLY"
        IF in_notification_cooldown(user_id):
            RETURN "THREAD_ONLY"
    RETURN recommendation
```

#### 9.5.5 UI Treatment of Proactive Messages

1. Proactive coach messages added while the user is away must appear in the main thread, not in a separate inbox surface.
2. The client UI must visually indicate that a proactive message arrived while the user was away.
3. This indication may be implemented with unread separators, unseen state, or a lightweight "while you were away" treatment.
4. Proactive messages should feel like part of the same coach relationship, not like system alerts from a separate subsystem.

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
| `sources` | `List<String>` | `["sessions","memory","program","episodic_date"]` | enabled retrieval sources |
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

Default embedding monthly budgets:

1. `memory_only = 0`
2. `standard = 5_000_000`
3. `premium_hybrid = 20_000_000`

### 10.3 Enforcement Rules

1. Policy checks must happen server-side.
2. Blocked features must degrade gracefully.
3. Budget enforcement must be auditable.
4. Rate limiting and concurrency admission decisions must be auditable.
5. Request admission metrics must be emitted by route, tier, and decision outcome.

---

## 11. Redaction and Risk Controls

### 11.1 Redaction Rules

1. Indexing must redact sensitive data classes before chunk persistence.
2. Raw events remain in audit store only under authorized access policies.
3. Retrieval must not return disallowed classes.

### 11.2 Minimal Risk Runtime Rules

1. Critical injury/emergency signals must trigger escalation behavior.
2. Risk events must be appended as `risk.*` events.
3. Unsafe recommendation paths should be interrupted or constrained by policy.

### 11.3 Redaction/Risk Audit Fields

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `safety_rule_version` | `String` | none | rule-set revision used |
| `redaction_rule_version` | `String` | none | redaction revision used |
| `decision` | `String` | none | allow/redact/block/escalate |
| `reason_codes` | `List<String>` | `[]` | deterministic justification codes |

---

## 12. Prompt Layers and Lifecycle

### 12.1 Purpose

The trainer should feel coherent, relational, and stable over time. That effect must come from explicit prompt layers and durable memory, not from hoping the model improvises the right personality on each run.

The implementation must separate:

1. app-owned runtime behavior,
2. trainer identity and tone,
3. trainer operating principles,
4. structured program state,
5. evolving user memory.

### 12.2 Prompt Layer Model

The system must use the following logical prompt layers.

| Layer | Scope | Ownership | Change Frequency | Purpose |
| --- | --- | --- | --- | --- |
| `system_prompt` | app-wide | product/runtime | rare | runtime rules, safety, tool usage, memory recall, delivery behavior, and trainer operating behavior |
| `coach_principles` | app-wide | product/coaching design | occasional | exercise design principles, progression rules, safety heuristics, coaching standards |
| `coach_soul` | per user | user-selected from curated options, then user-owned | rare | trainer identity, tone, relational stance, motivational style |
| `program_markdown` | per user | agent + system | ongoing | structured current training program and progression state |
| `memory_markdown` | per user | user + agent | ongoing | durable learned facts, recurring patterns, relational preferences |
| `episodic_notes` | per user | agent + system flushes | frequent/opportunistic | date-keyed notes from meaningful days or compaction/session-end flushes |

### 12.3 Layer Semantics

1. `system_prompt` is the runtime contract and must remain app-owned.
2. `coach_principles` is the domain guidance layer and should encode good coaching, exercise selection, progression, recovery, and safety principles.
3. `coach_soul` is the closest equivalent to OpenClaw's `SOUL.md`, but it is stored in the database as Markdown text and scoped per user.
4. `coach_soul` should make the trainer feel like a real person with a stable tone and relationship stance.
5. Trainer operating behavior that would otherwise live in a separate playbook should be part of `system_prompt`; the implementation should not maintain a separate `coach_playbook` layer.
6. `program_markdown` stores the user's current training program in structured Markdown and is updated over time as progress is made.
7. `memory_markdown` stores onboarding facts, preferences, injuries, equipment context, and other durable learned knowledge about the user.
8. `memory_markdown` and `episodic_notes` are the evolving continuity layers and should carry personalization over time.
9. The model must not freely rewrite `system_prompt` or `coach_principles` during normal operation.

### 12.4 Bootstrap and Onboarding Lifecycle

1. A new user should go through a bootstrap interview run rather than a static form-only setup.
2. The bootstrap interview should feel like a real trainer intake conversation.
3. The bootstrap prompt should instruct the trainer to ask about:
4. goals,
5. workout history and training age,
6. current routine,
7. injuries, pain, or medical constraints,
8. available equipment and workout locations,
9. schedule and lifestyle constraints,
10. motivation style and coaching preferences.
11. Responses from this bootstrap flow should be written into `memory_markdown`.
12. The user should choose or confirm a `coach_soul` at startup.
13. The bootstrap flow may also seed an initial `program_markdown` when enough training information is available.
14. Once onboarding is complete, bootstrap prompts should no longer run on every normal turn.

```pseudocode
FUNCTION run_bootstrap_interview(user_id: String) -> BootstrapResult:
    soul = select_or_confirm_coach_soul(user_id)
    interview = conduct_bootstrap_conversation(user_id, soul)
    memory_md = synthesize_memory_markdown(interview.answers)
    save_memory_markdown(user_id, memory_md)
    seed_initial_program(user_id, interview.answers)
    mark_bootstrap_complete(user_id)
    RETURN BootstrapResult(status="completed", coach_soul=soul)
```

### 12.5 Prompt Assembly Contract

At run time, prompt assembly should layer stable and evolving context in this order:

1. `system_prompt`
2. `coach_principles`
3. `coach_soul`
4. `program_markdown`
5. recent session/retrieval context
6. recalled `memory_markdown` / `episodic_notes` snippets as needed

Rules:

1. Stable layers should change slowly so the trainer remains coherent.
2. Evolving memory layers should personalize behavior without causing trainer identity drift.
3. `coach_soul` must be loaded every run for that user so the trainer feels like the same person over time.
4. `program_markdown` should be lightweight enough to inject directly or cheaply hydrate on every run.
5. Large evolving memory should be retrieved on demand rather than blindly injected in full.

### 12.6 Update Rules

1. `system_prompt` may only be changed by application deploy/config revision.
2. `coach_principles` may only be changed by curated product/coaching updates.
3. `coach_soul` may be selected at onboarding and later changed intentionally by the user.
4. `program_markdown` may be updated by the agent over time as progress is made, workouts are completed, and the plan evolves.
5. `memory_markdown` may be updated from onboarding, explicit user edits, or high-confidence structured agent updates.
6. `memory_markdown` may be updated when the user says to remember something or when the agent determines a durable preference/pattern should be stored.
7. `episodic_notes` may be created only when something meaningful happened or when a pre-compaction/session-end flush writes them.

### 12.7 Why This Produces a Human Feel

The trainer should feel human because:

1. identity is stable (`coach_soul`),
2. behavior is disciplined (`system_prompt` + `coach_principles`),
3. the user is known through durable memory (`memory_markdown`),
4. history accumulates (`memory_markdown` + `episodic_notes`).

This mirrors the useful OpenClaw pattern: stable persona plus evolving continuity, rather than letting all personality emerge from chat history alone.

---

## 13. Provider Adapter Architecture

### 13.1 Goal and Scope

1. Provider portability is in scope and required for the first architecture pass.
2. The implementation must support OpenAI, Anthropic, and Gemini through a stable runtime contract.
3. The core architecture (sessions, queueing, memory, indexing, retrieval, delivery) must not change when switching providers.

### 13.2 Adapter Layer Contract

The adapter layer must isolate provider differences in five areas:

1. request construction and model capability checks,
2. transcript hygiene and message ordering constraints,
3. tool schema/definition translation,
4. stream event normalization,
5. provider error classification and retryability.

### 13.3 Pseudocode Interfaces

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

### 13.4 Provider Capability Registry

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

### 13.5 Runtime Flow with Adapters

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

### 13.6 Non-Negotiable Invariants

1. Switching provider must not require schema changes in session, memory, or indexing tables.
2. Switching provider must preserve transcript append contracts and session concurrency behavior.
3. Provider-specific features may be enabled via capability flags, but unsupported features must degrade gracefully.
4. Errors must map to stable internal error classes used by retry/backoff policies.

---

## 14. Out of Scope (Current Version)

**Real-time collaborative multi-user program editing.** Concurrent multi-author real-time editing of `PROGRAM` is out of scope. Extension point: OT/CRDT collaboration layer on top of `memory_doc_versions`.

**Provider-specific product UX surfaces.** Provider-branded advanced UX controls are out of scope for first pass. Extension point: provider-specific admin/settings UI.

**Cross-tenant federated memory sharing.** Shared memory graphs across users/coaches are out of scope. Extension point: separate `shared_memory` domain with explicit ACLs.

**Automated medical diagnosis workflows.** Medical diagnostic automation is out of scope. Extension point: regulated safety escalation integration and clinician review workflow.

---

## 15. Current Defaults and Deferred Design Areas

### 15.1 Current Defaults

1. `sessionId` day-boundary rotation is enabled by default.
2. Idle expiry is enabled by default with `180` minutes, and reset policy remains configurable per user/plan.
3. New-session episodic-note reads default to `today_and_yesterday`, with larger read windows allowed by plan/user policy.
4. Pre-compaction and session-end memory flushes are silent background operations and do not notify the user by default.
5. Raw `session_events`, versioned documents, and indexed chunks are retained indefinitely by default.
6. Tree-linked session structure remains internal only and is not exposed as user-facing branch navigation.
7. Program edits do not require user confirmation by default.
8. Retry backoff should use exponential backoff with jitter.
9. Default embedding monthly budgets are:
10. `memory_only = 0`
11. `standard = 5_000_000`
12. `premium_hybrid = 20_000_000`

### 15.2 Deferred Design Areas

The following areas are intentionally deferred and must be specified in later design passes before production implementation:

1. queue topology, worker-role assignment, job payload schemas, dead-letter policy, and stalled-job recovery,
2. exact `program_markdown` structure and parsing model,
3. exact heartbeat candidate-reason taxonomy,
4. exact BFF/feed/tray payload schemas and client event contracts.

---

## 16. Definition of Done

### 16.1 API Gateway and Session Resolve

- [ ]  API validates auth, schema, and idempotency before mutation.
- [ ]  API rejects malformed request bodies, invalid ownership, invalid component actions, and malformed SSE resume cursors before enqueueing work.
- [ ]  API resolves canonical `sessionKey` and active `sessionId` for each inbound action.
- [ ]  API persists inbound event and enqueues worker jobs in one request flow.
- [ ]  API returns deterministic `202 Accepted` with `sessionKey`, `runId`, `jobId`, and stream URL.
- [ ]  API gateway enforces Redis-backed token bucket rate limits for message submissions.
- [ ]  API gateway enforces active-run and active-stream concurrency limits per effective user policy.
- [ ]  `429` responses include structured retry metadata and limit scope.

### 16.2 Transcript and Event Model

- [ ]  `session_events` is append-only and immutable after insert.
- [ ]  Parent linkage is enforced and leaf pointer advances atomically.
- [ ]  Event catalog supports `context_includable`, `indexable`, and `audit_only` flags.
- [ ]  Workout, program, memory, risk, and system event families are implemented.
- [ ]  Semantic tool failures are recorded as explicit structured events with agent-facing guidance.

### 16.3 Queue, Workers, and Concurrency

- [ ]  BullMQ jobs in Redis are durable enough for restart/recovery under configured persistence/HA policy.
- [ ]  Per-session advisory locks prevent concurrent mutation of same session.
- [ ]  Optimistic leaf/version guard rejects stale writes.
- [ ]  Idempotency and bounded retries prevent duplicate side effects.
- [ ]  Failed jobs are observable and routed to dead-letter handling.
- [ ]  Queue state is recoverable without data loss because canonical outcomes remain in Postgres.

### 16.4 Memory and Document Lifecycle

- [ ]  `MEMORY`, `PROGRAM`, and `EPISODIC_DATE` docs exist as versioned DB records.
- [ ]  Pre-compaction and session-end flush paths can create/update date-keyed episodic notes.
- [ ]  New-session memory reads follow configured window policy.
- [ ]  Memory writes are attributable to actor and run/session provenance.
- [ ]  `program_markdown` is updated over time as workout progress changes the user's plan.

### 16.5 Indexing and Retrieval

- [ ]  Dirty tracking with delta thresholds gates indexing work.
- [ ]  Session indexing extracts, redacts, chunks, embeds, and upserts with provenance.
- [ ]  Memory doc indexing reuses embedding cache via provider/model/config+chunk hash.
- [ ]  Redis hybrid retrieval (vector + FTS/BM25) returns ranked results with Postgres provenance.
- [ ]  Retrieval can be stale only within async sync window, with visible watermark metrics.
- [ ]  Full Redis index rebuild from Postgres is supported and documented.

### 16.6 Trainer Tools and Workout State

- [ ]  The trainer acts through a typed tool registry covering context recall, workout execution, live adjustment, program management, calculations, and memory writes.
- [ ]  `workout_generate` creates a workout session structure when the user is ready to train.
- [ ]  Exercise library definition, program prescription, and live workout execution state are represented as separate data layers.
- [ ]  The persistent workout tray is backed by live `WorkoutSessionState` rather than inline feed cards.

### 16.7 Delivery and Streaming

- [ ]  Streaming emits lifecycle and delta events during run execution.
- [ ]  SSE reconnect supports resume via `Last-Event-ID` or `since` cursor.
- [ ]  Missed events are replayed from durable `stream_events` before live tail.
- [ ]  Replay-window expiry returns deterministic `409 replay_window_expired`.
- [ ]  Final run output is durably persisted regardless of stream disconnect.
- [ ]  Outbox retries transient failures and records terminal failures.
- [ ]  Duplicate final deliveries are prevented by idempotency constraints.
- [ ]  Heartbeat runs are pre-gated by proactive feature enablement, quiet hours, active run/workout state, recency, and candidate-reason checks before any LLM call.
- [ ]  Heartbeat result contract supports `HEARTBEAT_OK`, `THREAD_ONLY`, and `THREAD_PLUS_PUSH`.
- [ ]  Server-side delivery policy can downgrade `THREAD_PLUS_PUSH` to `THREAD_ONLY`.
- [ ]  Proactive thread messages are visibly marked in the UI as having arrived while the user was away.

### 16.8 Client View Aggregation and UX Contract

- [ ]  The primary coach screen is driven by one aggregated backend payload rather than many client-side orchestration requests.
- [ ]  Aggregated client payloads combine canonical session/run/workout/feed state without creating alternate business truth.
- [ ]  The UI contract includes a persistent bottom dock with text entry, microphone control, and live transcript behavior.
- [ ]  The workout tray expands upward in place above the input dock and does not cover or replace the persistent input dock.
- [ ]  During workout-active states, the current exercise or rest state is controlled through the persistent workout tray, while inline feed UI remains reserved for contextual and reflective components.

### 16.9 Prompt Layers and Bootstrap Lifecycle

- [ ]  Prompt assembly separates runtime rules, coaching principles, coach identity, structured program state, and evolving memory into distinct layers.
- [ ]  `coach_soul` is stored per user as Markdown text and loaded on every run for that user.
- [ ]  New users go through a bootstrap interview flow that populates `memory_markdown`.
- [ ]  Bootstrap interview covers goals, training history, injuries/constraints, equipment, schedule, and coaching preferences.
- [ ]  `system_prompt` and `coach_principles` remain curated and are not freely rewritten during normal operation.
- [ ]  Evolving personalization is stored in `memory_markdown` and `episodic_notes`, while training structure evolves in `program_markdown`, instead of drifting the trainer identity layer.

### 16.10 Policy, Cost, and Safety Enforcement

- [ ]  Effective policy resolution merges global, plan, and user overrides.
- [ ]  Plan-tier knobs control indexing, retrieval, sources, and embedding budget.
- [ ]  Plan-tier knobs control request rate limits and concurrency admission limits.
- [ ]  Redaction rules execute before indexed storage.
- [ ]  High-risk injury/emergency signals trigger deterministic escalation/constraint behavior.
- [ ]  All policy, redaction, and high-risk escalation decisions are auditable.

### 16.11 Provider Adapter Portability

- [ ]  Provider adapter layer supports OpenAI, Anthropic, and Gemini via one stable runtime contract.
- [ ]  Transcript hygiene is provider-aware and deterministic before each model run.
- [ ]  Tool definitions are transformed per provider from one canonical internal tool schema.
- [ ]  Stream events are normalized to one internal event model used by delivery.
- [ ]  Provider errors map to stable retryable/non-retryable/internal classes.
- [ ]  Switching providers requires no changes to session, memory, indexing, or retrieval schemas.

### 16.12 Cross-Feature Parity Matrix

| Test Case | OpenAI | Anthropic | Gemini | Memory-Only Plan | Premium Hybrid Plan |
| --- | --- | --- | --- | --- | --- |
| Inbound user message accepted and persisted | [ ] | [ ] | [ ] | [ ] | [ ] |
| Per-session lock prevents concurrent head corruption | [ ] | [ ] | [ ] | [ ] | [ ] |
| Date-keyed episodic note flush on compaction cycle | [ ] | [ ] | [ ] | [ ] | [ ] |
| Session indexing obeys delta thresholds | [ ] | [ ] | [ ] | [ ] | [ ] |
| Hybrid retrieval returns traceable provenance | [ ] | [ ] | [ ] | [ ] | [ ] |
| Live workout tools mutate current session state without corrupting program/library layers | [ ] | [ ] | [ ] | [ ] | [ ] |
| Final delivery survives stream disconnect | [ ] | [ ] | [ ] | [ ] | [ ] |
| SSE reconnect replays missed events and resumes live | [ ] | [ ] | [ ] | [ ] | [ ] |
| Rate limiting returns structured `429` with retry metadata | [ ] | [ ] | [ ] | [ ] | [ ] |
| Heartbeat proactive message is added to thread and push-downgraded by server policy when needed | [ ] | [ ] | [ ] | [ ] | [ ] |
| Bootstrap flow seeds memory, program, and stable coach soul | [ ] | [ ] | [ ] | [ ] | [ ] |
| Semantic tool errors recover within run with structured guidance | [ ] | [ ] | [ ] | [ ] | [ ] |
| Budget limit blocks embedding with graceful degradation | [ ] | [ ] | [ ] | [ ] | [ ] |
| High-risk escalation path emits risk events | [ ] | [ ] | [ ] | [ ] | [ ] |
| Canonical tool definitions execute through adapter mappings | [ ] | [ ] | [ ] | [ ] | [ ] |

### 16.13 Integration Smoke Test

```
-- Setup
user_id = "u_123"
policy = load_effective_policy(user_id)
ASSERT policy is not NONE

-- Bootstrap
bootstrap = run_bootstrap_interview(user_id)
ASSERT bootstrap.status == "completed"
ASSERT bootstrap.coach_soul is not NONE
ASSERT load_memory_markdown(user_id) is not NONE

-- Inbound action
resp = POST /messages { user_id, text="Completed squat set 225x5 RPE 8", idempotency_key="k1" }
ASSERT resp.status == 202
ASSERT resp.session_key == "user:u_123:main"
ASSERT resp.job_id is not NONE
ASSERT resp.stream_url is not NONE

-- Gateway validation
bad = POST /messages { user_id, component_action={ type="bad_action" } }
ASSERT bad.status >= 400
ASSERT bad.status < 500

-- Rate limiting metadata
limited = POST /messages { user_id, text="spam", idempotency_key="k-limit" } after exhausting bucket
ASSERT limited.status == 429
ASSERT limited.error.scope is not NONE
ASSERT limited.error.retry_after_seconds > 0

-- BFF payload
surface = GET /coach-surface { user_id }
ASSERT surface.status == 200
ASSERT surface.payload.feed is not NONE
ASSERT surface.payload.workout is not NONE OR surface.payload.run is not NONE

-- Workout generation
generated = workout_generate(user_id, guidance={ ready_now: true })
ASSERT generated.workout_session_id is not NONE
ASSERT LENGTH(generated.exercises) > 0

-- Worker run and lock behavior
job = dequeue("agent.run_turn")
ASSERT job.session_key == "user:u_123:main"
lock_ok = pg_try_advisory_lock(hash(job.session_key))
ASSERT lock_ok == true

-- Transcript append
event = append_session_event(...)
ASSERT event.event_type == "conversation.user_message" OR event.event_type == "workout.set.completed"

-- Semantic tool recovery
tool_result = execute_tool_with_recovery(invalid_semantic_adjustment_call)
ASSERT tool_result.status == "semantic_error"
ASSERT tool_result.error.agent_guidance is not NONE

-- Live workout mutation
current = workout_get_current_state(user_id)
adjusted = workout_adjust_load(current.workout_session_id, delta=5)
ASSERT adjusted.current_exercise_id == current.current_exercise_id

-- Flush + memory write
flush_job = enqueue("memory.flush_session_end", { session_key: job.session_key })
process(flush_job)
episodic_doc = get_memory_doc(user_id, "EPISODIC_DATE", current_user_date(user_id))
ASSERT episodic_doc is not NONE

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

-- Heartbeat gating + delivery enforcement
hb_skip = maybe_run_heartbeat(user_id, now=quiet_hours_time)
ASSERT hb_skip.status == "skipped"

hb = run_heartbeat_once(user_id, now=active_hours_time)
ASSERT hb.result IN ["HEARTBEAT_OK", "THREAD_ONLY", "THREAD_PLUS_PUSH"]

delivery = enforce_heartbeat_delivery(user_id_with_push_disabled, "THREAD_PLUS_PUSH")
ASSERT delivery == "THREAD_ONLY"

proactive_msg = append_proactive_thread_message(user_id, "Coach message", delivery)
ASSERT proactive_msg.thread_visible == true
ASSERT proactive_msg.away_marker == true

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
| `risk.*` | `risk.alert` | `true` | `optional` | `false` |
| `system.*` | `system.retry` | `false` | `false` | `true` |
| `compaction.*` | `compaction.summary` | `true` | `true` | `false` |

---

## Appendix B: Design Decision Rationale

**Why Redis/BullMQ jobs instead of in-process only queues?** The PT app is multi-user SaaS oriented and requires durable retries, horizontal worker scaling, and restart-safe job orchestration.

**Why keep `sessionKey` with only one current user chat?** It preserves architectural extensibility and stable routing identity while allowing `sessionId` rotation without losing continuity.

**Why append-only events plus mutable session state?** This separates immutable audit history from operational mutable pointers (`leaf_event_id`, counters), improving traceability and conflict handling.

**Why SSE-first delivery with replay + outbox?** SSE gives responsive UX; replay via cursor handles mobile disconnects; outbox guarantees final consistency independent of connection state.

**Why dirty marking plus sync jobs for indexing?** It coalesces burst writes, controls embedding costs, and still guarantees eventual indexing via durable worker processing.
