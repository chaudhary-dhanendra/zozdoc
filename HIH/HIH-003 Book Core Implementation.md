# HIH-003 Book Core Implementation

## 1. Purpose

This chapter defines the implementation handbook for the `hermes-book` crate, the single-writer Book Core responsible for processing commands for one book partition.

The document is intentionally concrete enough for a Rust engineer or Codex agent to scaffold and implement the crate without inventing architecture.

The Book Core must preserve these non-negotiable properties:

- one mutable owner of book state per running book;
- book-local sequence assignment only;
- fixed-point integer arithmetic only;
- deterministic state transitions and deterministic replay;
- immutable `EngineEvent` records;
- hash-chained event append before externally visible success;
- no global sequencer;
- no database, Kafka, cloud service, or network dependency in the hot path;
- no heap allocation during steady-state matching;
- no locks inside `process_order`;
- bounded queues, bounded memory, fail-fast behaviour, and replay equivalence;
- all state mutation owned by `BookCore`.

## 2. Related HES Volumes

Related HES material remains the source of truth. This chapter maps those requirements into crate-level implementation boundaries without expanding HES.

- HES exchange overview: single-writer, active/passive, deterministic exchange core principles.
- HES book-core reference: order-book ownership, matching integration, event-first success semantics.
- HES risk reference: reservation boundary and deterministic reject semantics.
- HES replay reference: snapshot-plus-event recovery and replay equivalence.
- HES operations reference: bounded memory, observability, and fail-fast operations.
- HES ADRs: single-writer book core, book-local ordering, fixed-point arithmetic, event sourcing, no DB in hot path, no Kafka before decision, hash-chained events, deterministic replay, bounded lock-free rings, hot/cold path separation, and production readiness gates.

## 3. Implementation Scope

This chapter covers only `hermes-book` implementation material.

In scope:

- crate boundary and module layout;
- runtime loop for one book partition;
- command ingress and response egress contracts;
- book-local sequence allocation;
- order validation, risk, matching, clearing, event construction, append, response, and snapshot boundaries;
- memory ownership, arenas, pools, price levels, order nodes, event buffers, and ring interaction;
- deterministic replay and crash recovery;
- backpressure and `ENGINE_BUSY` behaviour;
- public traits, structs, enums, pseudocode, tests, benchmarks, and review gates.

Out of scope:

- production Rust source files;
- HES modifications;
- gateway, risk, matching, clearing, replay, or market-data chapter expansion;
- DOCX/PDF generation.

## 4. Non-Goals

The Book Core must not become a general exchange runtime.

Non-goals:

- no global sequencer;
- no cross-book matching inside this crate;
- no account ledger ownership;
- no database persistence API in `process_order`;
- no Kafka producer or consumer in the decision path;
- no async executor in the matching path;
- no floating point arithmetic;
- no JSON parsing in the hot path;
- no dynamic rule engine;
- no hidden background thread that mutates book state;
- no best-effort success before durable event append;
- no unbounded queues, maps, caches, or logs;
- no production `src/` files created by this handbook task.

## 5. Book Core Design Summary

`BookCore` is the exclusive mutable owner of a single book's state. External components send bounded commands to the core and receive bounded responses from the core. The core assigns a book-local sequence, validates the command, invokes risk, matching, and clearing boundaries in deterministic order, builds immutable engine events, appends those events to the hash-chained log, and only then publishes success responses.

The design separates hot and cold responsibilities:

- hot path: command decode from typed command, validation, sequence allocation, risk boundary, matching boundary, clearing boundary, event build, append, response publication, metrics counters;
- cold path: configuration loading, startup validation, snapshot file IO, replay orchestration, log rotation, benchmark harnesses, human-readable diagnostics.

Every state transition must be reproducible from a verified snapshot and verified event suffix. If an action cannot be represented by an `EngineEvent`, it must not mutate durable book state.

## 6. Crate Boundary

The concrete crate is `hermes-book`.

Allowed crate responsibilities:

- own `BookCore`, `BookCoreState`, and the single-writer runtime;
- define book command and response contracts;
- define boundary traits to risk, matching, clearing, events, snapshots, replay, and metrics;
- coordinate deterministic processing order;
- own memory pools used by the book state;
- provide test fixtures and benchmark harness definitions.

Forbidden crate responsibilities:

- implement account ledger storage;
- implement external gateway protocols;
- own global market-data dissemination;
- open SQL connections in hot modules;
- instantiate Kafka/cloud clients in hot modules;
- perform network IO inside `process_order`;
- use floats or wall-clock timestamps for ordering.

## 7. Module Layout

`hermes-book` is organized around explicit boundaries. Public exports come from `lib.rs`. Hot-path modules are small and dependency-restricted. IO adapters are outside the matching decision path.

Top-level modules:

- `core`: `BookCore` orchestration and command processing.
- `runtime`: single-writer loop, lifecycle, backpressure, shutdown.
- `command`: typed inbound commands.
- `response`: typed outbound responses.
- `state`: in-memory book state ownership.
- `config`: validated startup configuration.
- `error`: stable error and fatal error types.
- `metrics`: metrics trait and counters.
- `sequence`: book-local sequence allocator.
- `snapshot`: snapshot provider boundary.
- `replay`: replay engine boundary.
- `memory`: arenas, pools, and layout guidance.
- `pipeline`: validation, risk, matching, clearing, event build, publish steps.

## 8. File Layout

```text
crates/hermes-book/
├── Cargo.toml
├── src/
│   ├── lib.rs
│   ├── core.rs
│   ├── runtime.rs
│   ├── command.rs
│   ├── response.rs
│   ├── state.rs
│   ├── config.rs
│   ├── error.rs
│   ├── metrics.rs
│   ├── sequence.rs
│   ├── snapshot.rs
│   ├── replay.rs
│   ├── memory/
│   │   ├── mod.rs
│   │   ├── arena.rs
│   │   ├── pools.rs
│   │   ├── layout.rs
│   ├── pipeline/
│   │   ├── mod.rs
│   │   ├── validate.rs
│   │   ├── risk.rs
│   │   ├── matching.rs
│   │   ├── clearing.rs
│   │   ├── event.rs
│   │   ├── publish.rs
│   └── tests/
│       ├── deterministic.rs
│       ├── replay.rs
│       ├── property.rs
│       ├── performance.rs
```

### `Cargo.toml`

- Responsibility: declare `hermes-book` package metadata, feature flags, dependency versions, benchmark targets, and test-only dependencies.
- Public items: none.
- Forbidden behaviour: enabling default features that pull network clients, async runtimes, SQL drivers, or float-based numeric crates into hot modules.
- Dependencies allowed: `hermes-fixed`, `hermes-ids`, `hermes-events`, `hermes-matching` trait surface, `hermes-risk` trait surface, `hermes-clearing` trait surface, `proptest` as dev dependency, criterion-compatible benchmark tooling as dev dependency.
- Dependencies forbidden: SQL drivers, Kafka clients, HTTP servers, tracing exporters in hot path, random-number crates for production sequencing, float math crates.
- Tests required: dependency audit test or CI check that denies forbidden crates.

### `src/lib.rs`

- Responsibility: crate documentation, public re-exports, feature-gated test helpers.
- Public items: traits, command/response types, config, errors, core handles, sequence type, replay/snapshot interfaces.
- Forbidden behaviour: initialization side effects, global mutable state, logger installation, thread spawning.
- Dependencies allowed: sibling modules only.
- Dependencies forbidden: IO, network, filesystem, time, random.
- Tests required: public API compile tests and documentation examples that do not create production files.

### `src/core.rs`

- Responsibility: define `BookCore` and coordinate `process_command` and `process_order`.
- Public items: `BookCore`, `OrderProcessor`, `ProcessOrderContext`, `PipelineResult`.
- Forbidden behaviour: locks inside `process_order`, heap allocation after warmup, DB/Kafka/network IO, wall-clock ordering, response before append.
- Dependencies allowed: `state`, `sequence`, `pipeline`, `command`, `response`, `error`, `metrics`.
- Dependencies forbidden: filesystem snapshot IO, external protocol crates, async executor.
- Tests required: accepted/rejected order, event append failure, no response before append, no mutation without event.

### `src/runtime.rs`

- Responsibility: own the single-writer execution loop and lifecycle states.
- Public items: `BookCoreRuntime`, `BookRuntimeState`, `BackpressureDecision`.
- Forbidden behaviour: multiple writers for one book, unbounded polling buffers, blocking external services, hidden state mutation thread.
- Dependencies allowed: `core`, `command`, `response`, `metrics`, `snapshot` trigger interface.
- Dependencies forbidden: matching internals, SQL/Kafka, global scheduler.
- Tests required: drain, busy mode, shutdown, fatal transition, bounded queue handling.

### `src/command.rs`

- Responsibility: define typed inbound command enum after gateway normalization.
- Public items: `BookCoreCommand`, command payload structs, stable command IDs.
- Forbidden behaviour: parse JSON in hot path, allocate variable strings in steady state, use floats, use system time as ordering.
- Dependencies allowed: fixed-point types, IDs, side, time-in-force enums if fixed-width.
- Dependencies forbidden: gateway protocol parsers, serde JSON in hot path.
- Tests required: command parsing from typed fixture, invalid command rejection, duplicate client order ID mapping.

### `src/response.rs`

- Responsibility: define response enum and publication semantics.
- Public items: `BookCoreResponse`, response codes, response metadata.
- Forbidden behaviour: success without durable event reference, secrets in debug output, unbounded payloads.
- Dependencies allowed: IDs, sequence, event IDs, stable error codes.
- Dependencies forbidden: protocol-specific encoders, network clients.
- Tests required: success response includes appended event identity; reject response is deterministic.

### `src/state.rs`

- Responsibility: own in-memory price levels, order indexes, and replay-apply state.
- Public items: `BookCoreState`, `BookStateStore`, state hash helper.
- Forbidden behaviour: state mutation by external component, heap allocation during matching, non-deterministic map iteration, negative quantities.
- Dependencies allowed: memory arenas/pools, fixed types, IDs.
- Dependencies forbidden: event appenders, response sinks, system time.
- Tests required: insert/cancel/replace/fill invariants and replay state hash equivalence.

### `src/config.rs`

- Responsibility: define validated configuration.
- Public items: `BookCoreConfig`, config validation errors.
- Forbidden behaviour: silently default unsafe limits, allow zero queue capacity, accept float scales.
- Dependencies allowed: error types and fixed constants.
- Dependencies forbidden: runtime mutation from env after startup.
- Tests required: invalid config rejection and maximum bound enforcement.

### `src/error.rs`

- Responsibility: stable recoverable and fatal error enums.
- Public items: `BookCoreError`, `BookCoreFatalError`, stable machine codes.
- Forbidden behaviour: leaking account secrets, converting fatal append failure into success, string-only errors.
- Dependencies allowed: IDs and sequence for context.
- Dependencies forbidden: anyhow in public API for hot path.
- Tests required: error mapping and fatal/non-fatal classification.

### `src/metrics.rs`

- Responsibility: define metrics counters, gauges, histograms, and recorder boundary.
- Public items: `BookCoreMetrics`, `MetricsRecorder`.
- Forbidden behaviour: allocating labels per command, locking in inner matching loop, logging every match in steady state.
- Dependencies allowed: atomics for cold publication, fixed label sets.
- Dependencies forbidden: network exporters in hot path.
- Tests required: counter increments and no allocation assertion for hot metric recording.

### `src/sequence.rs`

- Responsibility: allocate monotonic book-local sequences.
- Public items: `BookSequence`, `SequenceAllocator`.
- Forbidden behaviour: global sequence calls, random sequences, wall-clock-derived sequences, wraparound.
- Dependencies allowed: primitive integer types and error types.
- Dependencies forbidden: database counters, distributed lock services.
- Tests required: monotonicity, overflow fatal error, replay reset from snapshot.

### `src/snapshot.rs`

- Responsibility: describe snapshot triggers and provider interface.
- Public items: `SnapshotProvider`, `SnapshotTrigger`, snapshot metadata.
- Forbidden behaviour: mutate live state from snapshot writer, produce snapshot without last event hash, unbounded serialization buffer.
- Dependencies allowed: state read views, sequence, event hash.
- Dependencies forbidden: hot-path filesystem calls from `process_order`.
- Tests required: trigger policy, snapshot metadata, restore validation.

### `src/replay.rs`

- Responsibility: replay snapshot plus hash-verified events into state.
- Public items: `ReplayEngine`, `ReplayMode`.
- Forbidden behaviour: consult live risk service during replay, use current wall clock, skip hash verification.
- Dependencies allowed: state, event types, snapshot interfaces.
- Dependencies forbidden: command ingress, response publication, network IO.
- Tests required: replay equivalence, hash mismatch rejection, crash boundary cases.

### `src/memory/mod.rs`

- Responsibility: memory subsystem exports and documentation.
- Public items: arena/pool/layout re-exports.
- Forbidden behaviour: expose mutable aliases to live state.
- Dependencies allowed: `arena`, `pools`, `layout`.
- Dependencies forbidden: pipeline or runtime dependencies.
- Tests required: pool exhaustion and recycle tests.

### `src/memory/arena.rs`

- Responsibility: preallocated arenas for order nodes and level storage.
- Public items: arena handles, capacity errors.
- Forbidden behaviour: grow-on-demand in steady state, allocate after warmup, return dangling handles.
- Dependencies allowed: fixed capacity collections if audited.
- Dependencies forbidden: `Vec::push` growth in hot path unless capacity was pre-reserved and checked.
- Tests required: capacity bounds, reuse, no allocation in matching benchmark.

### `src/memory/pools.rs`

- Responsibility: object pools for reusable order nodes, fill buffers, and event scratch buffers.
- Public items: pool handles, lease types.
- Forbidden behaviour: leaking leases, blocking waits, unbounded fallback allocation.
- Dependencies allowed: local arrays and indexes.
- Dependencies forbidden: mutex-protected global pools in `process_order`.
- Tests required: exhaustion, return, double-free detection in debug builds.

### `src/memory/layout.rs`

- Responsibility: cache-line and field-layout guidance.
- Public items: layout marker types and constants.
- Forbidden behaviour: hiding semantic state in padding, relying on undefined layout unless `repr` is justified.
- Dependencies allowed: core memory constants.
- Dependencies forbidden: unsafe layout tricks without policy approval.
- Tests required: size/alignment assertions for hot structs.

### `src/pipeline/mod.rs`

- Responsibility: compose pipeline stage modules.
- Public items: pipeline stage traits and result re-exports.
- Forbidden behaviour: perform work directly that belongs to stage modules.
- Dependencies allowed: stage siblings.
- Dependencies forbidden: runtime loop ownership.
- Tests required: pipeline order test with instrumented boundaries.

### `src/pipeline/validate.rs`

- Responsibility: deterministic command/order validation before mutation.
- Public items: `validate_order_boundary` helper and validation error codes.
- Forbidden behaviour: reserve risk, mutate state, append events, or call wall clock.
- Dependencies allowed: command, config, fixed types.
- Dependencies forbidden: state mutation, event appender.
- Tests required: invalid price/quantity/side/time-in-force rejection.

### `src/pipeline/risk.rs`

- Responsibility: invoke risk reservation boundary.
- Public items: `RiskBoundary`.
- Forbidden behaviour: mutate book state before accepted reservation, network call, non-deterministic retry loop.
- Dependencies allowed: process context, risk view, stable reject codes.
- Dependencies forbidden: gateway and database clients.
- Tests required: risk accept/reject and deterministic error mapping.

### `src/pipeline/matching.rs`

- Responsibility: invoke matching boundary against owned state.
- Public items: `MatchingBoundary`.
- Forbidden behaviour: allocate heap memory, lock, call external services, emit response directly.
- Dependencies allowed: mutable `BookCoreState`, fixed types, preallocated scratch buffers.
- Dependencies forbidden: event appender, response sink, risk network adapters.
- Tests required: full fill, partial fill, cancel, replace, no negative open quantity.

### `src/pipeline/clearing.rs`

- Responsibility: derive clearing deltas from deterministic matching output.
- Public items: `ClearingBoundary`.
- Forbidden behaviour: post ledger entries directly, use floats, reorder fills.
- Dependencies allowed: fills, fixed quantities, account IDs.
- Dependencies forbidden: wallet database clients.
- Tests required: maker/taker delta construction and conservation checks.

### `src/pipeline/event.rs`

- Responsibility: build immutable `EngineEvent` payloads and append them.
- Public items: `EngineEventBuilder`, `EventAppender`.
- Forbidden behaviour: publish response before append, mutate event after hash, omit previous hash, use random hash salt.
- Dependencies allowed: event crate, sequence, previous hash, pipeline result.
- Dependencies forbidden: response sink except through publish stage.
- Tests required: stable serialization, hash chain, append-failure fatal behaviour.

### `src/pipeline/publish.rs`

- Responsibility: publish bounded responses after append.
- Public items: `BookResponseSink` helpers.
- Forbidden behaviour: publish success without event ID, block indefinitely, allocate unbounded response payload.
- Dependencies allowed: response type and bounded sink.
- Dependencies forbidden: event construction mutation.
- Tests required: response-after-append ordering and full response queue handling.

### `src/tests/*.rs`

- Responsibility: crate-level deterministic, replay, property, and performance tests.
- Public items: none.
- Forbidden behaviour: relying on test execution order, random seeds without recording, floats.
- Dependencies allowed: dev-only test tools.
- Dependencies forbidden: real Kafka, SQL, cloud, or wall-clock ordering dependencies.
- Tests required: see sections 47-54.

## 9. Runtime Thread Model

One OS thread owns one active book writer. The runtime may pin the thread according to deployment configuration, but correctness must not depend on CPU pinning. A passive replica may exist outside this active writer, but it must replay immutable events and must not share mutable state with the active writer.

Runtime rules:

- exactly one `BookCore` mutates a given book state at a time;
- command ingress is via bounded source interface;
- response egress is via bounded sink interface;
- event append is synchronous with respect to success publication;
- snapshot work must use a consistent read view or explicit stop-the-world boundary;
- metrics export must not block matching.

## 10. Single-Writer Execution Loop

The execution loop repeatedly polls bounded commands, processes each command to completion or rejection, evaluates snapshot triggers, and handles shutdown.

Loop invariants:

- no command is processed concurrently with another command for the same book;
- sequence assignment happens inside the loop;
- durable append boundary is reached before success response;
- fatal error moves runtime into `Fatal` or `Quarantined` state;
- backpressure is decided by bounded queue state and configured thresholds.

## 11. Command Ingress Model

Commands arrive as typed `BookCoreCommand` values. Gateway parsing and authentication occur before this boundary.

Ingress requirements:

- bounded queue capacity configured at startup;
- non-blocking or bounded-time polling;
- stable command identity for idempotency;
- fixed-width IDs and fixed-point price/quantity fields;
- no unbounded strings in the command body;
- reject malformed commands before state mutation.

## 12. Response Egress Model

Responses leave through `BookResponseSink` after processing. A success response must reference the appended event sequence and event hash. A reject response must include a stable reject code and must not imply durable state mutation unless an event was appended.

Response rules:

- `Accepted`, `Filled`, `PartiallyFilled`, `Cancelled`, and `Replaced` require durable event identity;
- `Rejected` before mutation does not require event identity unless policy records reject events;
- response queue full must not silently drop success;
- response publication failure after append is recoverable by replay/query because the event is durable.

## 13. Book-Local Sequence Assignment

`BookSequence` is local to one book. It starts from snapshot metadata or configured genesis and increments by one for each command that reaches the sequenced processing boundary.

Rules:

- no global sequence allocation;
- no gaps for commands accepted into sequenced processing unless gap events are explicitly appended;
- overflow is fatal;
- replay restores next sequence from last applied event;
- sequence type is fixed-width unsigned integer with explicit serialization.

## 14. Order Processing Pipeline

Canonical order:

1. poll command;
2. classify command;
3. validate command boundary;
4. assign book sequence;
5. build `ProcessOrderContext`;
6. invoke risk boundary;
7. invoke matching boundary;
8. invoke clearing boundary;
9. build engine event;
10. append engine event and update hash-chain head;
11. publish response;
12. update metrics and maybe trigger snapshot.

No stage may skip ahead. In particular, response publication cannot precede event append.

## 15. Risk Invocation Boundary

Risk is a deterministic boundary. Book Core passes immutable context and receives accept/reject/reservation metadata.

Risk boundary requirements:

- no book state mutation before successful risk result for place/replace that increases exposure;
- risk reject maps to deterministic reject response;
- boundary must be local/in-memory or otherwise bounded and deterministic in the writer path;
- timeout-style non-determinism is forbidden inside `process_order`;
- risk result included in event material where relevant.

## 16. Matching Invocation Boundary

Matching mutates only `BookCoreState` owned by the writer. It consumes preallocated scratch buffers and returns deterministic match outcomes.

Matching boundary requirements:

- price-time priority must be deterministic;
- no heap allocation during steady-state matching;
- no locks;
- no external IO;
- fixed-point arithmetic only;
- output fill order exactly matches replay order.

## 17. Clearing Invocation Boundary

Clearing derives immutable deltas from matching outcomes. Book Core does not own the ledger, but it constructs or receives clearing deltas required for the event.

Clearing boundary requirements:

- deltas use fixed-point integers;
- deltas preserve asset and quantity conservation;
- fill order is unchanged;
- no ledger database write in hot path;
- errors before event append prevent success response.

## 18. EngineEvent Construction Boundary

`EngineEventBuilder` converts pipeline result into an immutable event candidate.

Event construction requirements:

- include book ID, book sequence, command ID, deterministic payload, previous event hash, and schema version;
- serialization is canonical;
- event hash is computed after canonical serialization;
- event is immutable after hash calculation;
- no wall-clock field participates in deterministic ordering.

## 19. Event Append Boundary

`EventAppender` appends the event to the hash-chained event log before success is externally visible.

Append rules:

- append failure after matching computation is fatal or quarantine, not success;
- hash-chain mismatch is fatal;
- appender returns durable event identity;
- append happens before response publication;
- appender is injected as a trait to keep hot logic testable.

## 20. Response Publication Boundary

`BookResponseSink` publishes responses after append. If publishing fails after append, the command result remains durable and can be recovered by event replay or client retry.

Publication rules:

- success includes event ID, sequence, and hash;
- duplicate retry after unknown response maps to existing durable result;
- full response queue triggers configured backpressure handling;
- no blocking forever.

## 21. Snapshot Trigger Boundary

Snapshot triggering is evaluated outside the inner matching stage. A snapshot may be triggered by event count, byte count, elapsed monotonic tick supplied by runtime, or explicit admin command.

Snapshot rules:

- snapshot metadata includes last sequence and last event hash;
- snapshot buffer is bounded;
- snapshot cannot observe partially-applied command state;
- live writer either pauses at a safe boundary or copies from a deterministic read view.

## 22. Replay Boundary

Replay rebuilds state from snapshot plus verified event suffix. It must not call live risk, gateway, matching randomness, or wall-clock ordering.

Replay rules:

- verify snapshot metadata before applying suffix;
- verify each event hash and previous hash;
- apply events in sequence order;
- compare final state hash when expected hash is available;
- never publish client responses during replay.

## 23. Memory Ownership Model

`BookCore` owns all mutable state required for matching. External modules receive references only through stage boundaries.

Ownership rules:

- `BookCoreState` is mutable only inside the writer thread;
- arenas own order nodes and return stable handles, not shared references;
- pools own scratch buffers and leases must not escape command processing;
- event buffers are reused only after append stage completes;
- snapshots receive immutable views or explicit copies.

## 24. Object Lifecycle

Order lifecycle:

1. command accepted into validation;
2. risk reservation accepted;
3. order node allocated or existing node located;
4. matching consumes, inserts, cancels, or replaces node;
5. clearing deltas created;
6. event appended;
7. response published;
8. filled/cancelled node returned to pool after event-safe mutation.

A node must never be visible in live book state unless its mutation can be reconstructed from an appended event.

## 25. Arena Allocation Strategy

Arenas are preallocated at startup according to validated config.

Arena rules:

- capacity is explicit and bounded;
- allocation failure maps to deterministic `ENGINE_BUSY` or capacity reject before mutation;
- no grow-on-demand in steady state;
- handles contain generation counters in debug or production if affordable;
- arena reset occurs only during controlled replay/bootstrap.

## 26. Object Pooling Strategy

Pools provide reusable objects for order nodes, fill vectors, event scratch buffers, and response scratch records.

Pool rules:

- pool exhaustion must be observable;
- no global mutex pool in `process_order`;
- pool leases are stack-scoped to command processing;
- buffers are cleared deterministically before reuse;
- benchmark asserts zero allocation after warmup.

## 27. Price Level Memory Model

Price levels are stored in deterministic order. Bids and asks may use separate fixed-capacity trees, skip lists, or indexed arrays, but iteration order must be canonical.

Price level invariants:

- one level per price per side;
- FIFO order within a level for equal priority;
- empty levels are removed or marked free deterministically;
- level handles are not reused until no order references remain;
- no non-deterministic hash-map iteration in matching.

## 28. Order Node Memory Model

Order nodes store hot fields contiguously and cold metadata separately when possible.

Hot fields:

- order ID;
- account/trader ID handle;
- side;
- price ticks;
- open quantity lots;
- priority sequence;
- next/previous node handles.

Node invariants:

- open quantity is never negative;
- filled plus open quantity never exceeds original quantity;
- node belongs to exactly one level or free list;
- external code cannot hold mutable node references.

## 29. Event Buffer Memory Model

Event construction uses bounded scratch buffers sized by maximum fills and delta count.

Event buffer rules:

- no unbounded serialization allocation;
- overflow maps to fatal config error or deterministic capacity reject before mutation;
- canonical byte representation is stable;
- hash uses canonical bytes only;
- scratch memory is cleared or overwritten before reuse.

## 30. Ring Buffer Interaction

Ingress and egress rings are bounded. The Book Core must understand full, empty, closed, and backpressured states.

Ring rules:

- command source returns at most configured batch size;
- response sink never blocks indefinitely;
- queue depth contributes to `BackpressureDecision`;
- `ENGINE_BUSY` is deterministic for queue-full conditions;
- no unbounded side queue is allowed.

## 31. Risk Cache View

Risk view is a bounded representation of reservations available to the writer.

Risk cache rules:

- keyed by fixed account and instrument IDs;
- updated only through deterministic boundary inputs;
- reject if required risk data is missing and policy requires fail-closed;
- no remote lookup inside `process_order`;
- replay uses event data, not live risk cache calls.

## 32. Clearing Delta View

Clearing delta view contains deterministic asset deltas derived from matches.

Clearing rules:

- deltas are immutable once included in event;
- sum checks run before append;
- fixed-point integer units only;
- no ledger posting side effect in Book Core;
- replay applies event interpretation, not external ledger state.

## 33. Snapshot Buffer View

Snapshot buffer view exposes the minimum state required for deterministic restore.

Snapshot contents:

- book ID;
- config fingerprint;
- last book sequence;
- last event hash;
- price levels and order nodes in canonical order;
- risk/clearing cache versions if required for deterministic validation;
- state hash.

## 34. Failure Handling

Failure classes:

- validation reject: no mutation, deterministic response;
- risk reject: no book mutation, deterministic response;
- capacity reject: no mutation, `ENGINE_BUSY` or stable reject;
- matching invariant violation: fatal quarantine;
- clearing invariant violation: fatal quarantine unless no mutation occurred and safe rollback is proven;
- event append failure: fatal quarantine before success response;
- response publication failure after append: durable success, recoverable egress failure.

## 35. Crash Recovery

Recovery starts from the newest verified snapshot and verified event suffix.

Crash cases:

- crash before append: no event exists; client retry is processed normally or rejected as duplicate only if prior durable event exists;
- crash after append before response: event exists; replay restores state and duplicate retry returns durable result;
- crash during snapshot: discard incomplete snapshot and use prior verified snapshot;
- crash during append: verify hash chain and appender commit marker before treating event as durable.

## 36. Deterministic Replay

Replay equivalence means live processing and replay produce the same state hash, sequence head, and event hash head.

Determinism requirements:

- no floats;
- no randomized iteration;
- no wall-clock decisions;
- no live risk calls;
- no race-dependent state;
- canonical serialization;
- stable overflow handling.

## 37. Backpressure and ENGINE_BUSY Behaviour

Backpressure protects bounded memory.

`BackpressureDecision` values:

- `Accept`: process normally;
- `ShedNew`: reject new command with `ENGINE_BUSY` before sequence if ingress is saturated;
- `DrainOnly`: stop accepting new commands and drain accepted commands;
- `EnterBusyMode`: publish busy status and keep polling administrative shutdown/recovery commands;
- `Fatal`: transition to fatal state for invariant or capacity corruption.

`ENGINE_BUSY` must be fast, deterministic, and non-mutating.

## 38. Observability

Observability must expose enough data to operate the writer without entering the hot path with heavyweight dependencies.

Required signals:

- processed command count;
- accepted/rejected counts by stable code;
- last assigned sequence;
- last durable sequence;
- last event hash prefix;
- ingress and egress queue depth;
- snapshot age and status;
- replay position;
- fatal state reason;
- allocation/pool exhaustion count.

## 39. Metrics

`BookCoreMetrics` stores hot counters and gauges in fixed locations. Exporters may read snapshots outside `process_order`.

Metric rules:

- fixed label cardinality;
- no allocation per command;
- no locks inside matching;
- latency histograms use preallocated buckets;
- counters distinguish validation reject, risk reject, capacity reject, append failure, and response failure.

## 40. Logging

Logging is structured and bounded.

Logging rules:

- no per-fill info log in steady state;
- fatal and recovery logs include book ID, sequence, event hash, and stable code;
- logs do not include secrets, raw credentials, or full account PII;
- debug logs in hot path are compile-time gated or sampled outside matching;
- append failure is always logged before quarantine.

## 41. Tracing

Tracing spans may wrap command processing but must not allocate dynamically in the inner loop.

Tracing rules:

- span fields use fixed keys;
- high-cardinality IDs are opt-in or sampled;
- tracing exporter is outside hot path;
- replay tracing is disabled by default unless debugging;
- trace state does not affect decisions.

## 42. Configuration

`BookCoreConfig` is validated before runtime starts.

Required fields:

- book ID and instrument ID;
- price scale and quantity scale identifiers;
- maximum active orders;
- maximum price levels per side;
- maximum fills per command;
- ingress and egress queue capacity;
- command batch size;
- snapshot trigger policy;
- event append durability policy;
- busy thresholds;
- feature flags with deterministic defaults.

Invalid config fails startup.

## 43. Error Types

Errors are stable, typed, and machine-actionable.

Recoverable examples:

- `InvalidCommand`;
- `InvalidPrice`;
- `InvalidQuantity`;
- `DuplicateCommand`;
- `RiskRejected`;
- `EngineBusy`;
- `UnknownOrder`;
- `CapacityExceeded`.

Fatal examples:

- `SequenceOverflow`;
- `ArithmeticOverflow`;
- `InvariantViolation`;
- `EventAppendFailed`;
- `HashChainMismatch`;
- `ReplayDiverged`;
- `SnapshotInvalid`;
- `ConfigurationInvalid`.

## 44. Public Traits

Trait contracts are implementation-grade. Method names are guidance, not committed production source.

### `BookCoreRuntime`

Purpose: own lifecycle of one single-writer core.

```rust
pub trait BookCoreRuntime {
    fn run_book_core_loop(&mut self) -> Result<(), BookCoreFatalError>;
    fn request_shutdown(&mut self, drain: bool);
    fn runtime_state(&self) -> BookRuntimeState;
}
```

Ownership model: owns or mutably borrows `BookCore`, command source, response sink, event appender, snapshot provider, and metrics recorder.

Error behaviour: fatal errors stop the loop and quarantine the writer.

Determinism: runtime polling cannot reorder commands within a source batch.

Test requirements: shutdown, drain, fatal transition, busy mode.

### `BookCommandSource`

Purpose: bounded source of typed commands.

```rust
pub trait BookCommandSource {
    fn poll_command(&mut self) -> Result<Option<BookCoreCommand>, BookCoreError>;
    fn poll_batch(&mut self, out: &mut [BookCoreCommand]) -> Result<usize, BookCoreError>;
    fn depth(&self) -> usize;
    fn capacity(&self) -> usize;
}
```

Ownership model: runtime has exclusive mutable access while polling.

Error behaviour: closed source leads to shutdown or busy policy; corrupt source is fatal if command order cannot be trusted.

Determinism: preserves FIFO order from gateway-normalized stream.

Test requirements: empty, full, closed, batch order.

### `BookResponseSink`

Purpose: bounded publication of responses.

```rust
pub trait BookResponseSink {
    fn try_publish(&mut self, response: BookCoreResponse) -> Result<(), BookCoreError>;
    fn depth(&self) -> usize;
    fn capacity(&self) -> usize;
}
```

Ownership model: runtime has exclusive mutable access during publication.

Error behaviour: full sink after append records egress failure; it must not undo durable event.

Determinism: publication status does not change book state.

Test requirements: success after append, queue full, no success before event.

### `BookCoreTrait`

Purpose: public processing interface for tests and runtime. The production struct is also named `BookCore`; this trait may be named `BookCoreApi` in code to avoid conflict.

```rust
pub trait BookCoreTrait {
    fn process_command(&mut self, command: BookCoreCommand) -> Result<BookCoreResponse, BookCoreError>;
    fn state_hash(&self) -> StateHash;
    fn last_sequence(&self) -> BookSequence;
}
```

Ownership model: requires `&mut self` to preserve single-writer state mutation.

Error behaviour: recoverable rejects return stable responses; fatal conditions are promoted by runtime.

Determinism: same initial state plus same command sequence yields same state hash.

Test requirements: command sequence golden tests and replay equivalence.

### `OrderProcessor`

Purpose: process order-like commands inside writer.

```rust
pub trait OrderProcessor {
    fn process_order(&mut self, ctx: ProcessOrderContext) -> Result<PipelineResult, BookCoreError>;
    fn process_cancel(&mut self, ctx: ProcessOrderContext) -> Result<PipelineResult, BookCoreError>;
    fn process_replace(&mut self, ctx: ProcessOrderContext) -> Result<PipelineResult, BookCoreError>;
}
```

Ownership model: owns mutable state and injected boundaries.

Error behaviour: errors before event append must leave no durable mutation unless represented by event.

Determinism: command type dispatch order is stable.

Test requirements: place, cancel, replace, full fill, partial fill.

### `BookStateStore`

Purpose: in-memory state operations.

```rust
pub trait BookStateStore {
    fn insert_order(&mut self, node: OrderNodeDraft) -> Result<OrderHandle, BookCoreError>;
    fn cancel_order(&mut self, order_id: OrderId) -> Result<OrderNodeSnapshot, BookCoreError>;
    fn replace_order(&mut self, order_id: OrderId, new_qty: Qty, new_price: Price) -> Result<OrderHandle, BookCoreError>;
    fn state_hash(&self) -> StateHash;
}
```

Ownership model: mutable only inside `BookCore`.

Error behaviour: capacity errors must occur before partial mutation or must rollback deterministically.

Determinism: iteration and hashing canonical.

Test requirements: invariants and state hash stability.

### `SequenceAllocator`

Purpose: allocate book-local sequences.

```rust
pub trait SequenceAllocator {
    fn assign_book_sequence(&mut self) -> Result<BookSequence, BookCoreFatalError>;
    fn peek_next(&self) -> BookSequence;
    fn restore_next(&mut self, next: BookSequence) -> Result<(), BookCoreFatalError>;
}
```

Ownership model: owned by writer.

Error behaviour: overflow is fatal.

Determinism: increments by one; no global calls.

Test requirements: monotonicity, overflow, restore.

### `RiskBoundary`

Purpose: deterministic risk reservation boundary.

```rust
pub trait RiskBoundary {
    fn invoke_risk_boundary(&mut self, ctx: &ProcessOrderContext) -> Result<RiskDecision, BookCoreError>;
}
```

Ownership model: receives immutable context and may update only its own bounded reservation cache if injected as writer-owned state.

Error behaviour: reject maps to stable response; unavailable required data fails closed.

Determinism: same context and cache snapshot produce same result.

Test requirements: accept, reject, missing data.

### `MatchingBoundary`

Purpose: mutate book state and produce fills.

```rust
pub trait MatchingBoundary {
    fn invoke_matching_boundary(
        &mut self,
        state: &mut BookCoreState,
        ctx: &ProcessOrderContext,
        risk: &RiskDecision,
    ) -> Result<MatchingResult, BookCoreError>;
}
```

Ownership model: exclusive mutable access to state during call.

Error behaviour: invariant violations are fatal through mapping.

Determinism: price-time priority and fill order canonical.

Test requirements: fill/cancel/replace/no allocation.

### `ClearingBoundary`

Purpose: derive clearing deltas.

```rust
pub trait ClearingBoundary {
    fn invoke_clearing_boundary(
        &mut self,
        ctx: &ProcessOrderContext,
        matching: &MatchingResult,
    ) -> Result<ClearingResult, BookCoreError>;
}
```

Ownership model: no ownership of ledger; returns immutable deltas.

Error behaviour: conservation failure is fatal or pre-append reject with proven no mutation.

Determinism: preserves matching fill order.

Test requirements: conservation and fixed-point deltas.

### `EngineEventBuilder`

Purpose: build canonical immutable events.

```rust
pub trait EngineEventBuilder {
    fn build_engine_event(
        &mut self,
        ctx: &ProcessOrderContext,
        result: &PipelineResult,
        previous_hash: EventHash,
    ) -> Result<EngineEvent, BookCoreError>;
}
```

Ownership model: may use bounded scratch buffer owned by writer.

Error behaviour: serialization overflow fails before append and before response.

Determinism: canonical serialization and hash.

Test requirements: golden bytes and hash stability.

### `EventAppender`

Purpose: durable append of hash-chained event.

```rust
pub trait EventAppender {
    fn append_event_before_response(&mut self, event: EngineEvent) -> Result<AppendedEvent, BookCoreFatalError>;
    fn last_hash(&self) -> EventHash;
}
```

Ownership model: exclusive mutable append handle for one book stream.

Error behaviour: append failure is fatal/quarantine.

Determinism: preserves event order and verifies previous hash.

Test requirements: append-before-response and hash mismatch.

### `SnapshotProvider`

Purpose: create and restore snapshots.

```rust
pub trait SnapshotProvider {
    fn should_snapshot(&self, trigger: SnapshotTrigger, metrics: &BookCoreMetrics) -> bool;
    fn write_snapshot(&mut self, state: &BookCoreState, last: AppendedEvent) -> Result<(), BookCoreError>;
    fn restore_latest(&mut self) -> Result<Option<BookSnapshot>, BookCoreFatalError>;
}
```

Ownership model: called at safe boundaries only.

Error behaviour: snapshot write failure is observable; invalid restore is fatal.

Determinism: snapshot canonical order.

Test requirements: trigger and restore.

### `ReplayEngine`

Purpose: recover state from snapshot and events.

```rust
pub trait ReplayEngine {
    fn replay_book_core_from_snapshot(
        &mut self,
        snapshot: BookSnapshot,
        events: EventStream,
        mode: ReplayMode,
    ) -> Result<BookCoreState, BookCoreFatalError>;
}
```

Ownership model: owns replay state; does not publish responses.

Error behaviour: hash mismatch or divergence is fatal.

Determinism: replay state hash equals live state hash.

Test requirements: snapshot+events, crash boundaries, duplicate retry.

### `MetricsRecorder`

Purpose: record bounded metrics.

```rust
pub trait MetricsRecorder {
    fn record_command_started(&mut self, sequence: Option<BookSequence>);
    fn record_command_finished(&mut self, response: &BookCoreResponse);
    fn record_error(&mut self, error: &BookCoreError);
    fn record_fatal(&mut self, error: &BookCoreFatalError);
}
```

Ownership model: writer-owned recorder or non-blocking sink.

Error behaviour: metrics failure must not affect matching decisions.

Determinism: metrics do not change state.

Test requirements: counter changes and no allocation.

## 45. Core Structs and Enums

Rust-style shapes below are contracts, not production files.

### `BookCore`

```rust
pub struct BookCore<R, M, C, E, S, Q> {
    pub config: BookCoreConfig,
    pub state: BookCoreState,
    pub sequence: BookSequence,
    pub risk: R,
    pub matching: M,
    pub clearing: C,
    pub event_builder: E,
    pub snapshot: S,
    pub metrics: BookCoreMetrics,
    pub runtime_state: BookRuntimeState,
    pub scratch: Q,
}
```

Purpose: exclusive mutable owner for one book.

Invariants: one writer; state hash matches event head after append; no success before append.

Ownership rules: constructed at startup or replay restore; not `Clone`; not shared mutably.

Serialization: live struct is not serialized; snapshots serialize state and metadata only.

### `BookCoreConfig`

```rust
pub struct BookCoreConfig {
    pub book_id: BookId,
    pub instrument_id: InstrumentId,
    pub price_scale: ScaleId,
    pub quantity_scale: ScaleId,
    pub max_active_orders: u32,
    pub max_price_levels_per_side: u32,
    pub max_fills_per_command: u16,
    pub ingress_capacity: u32,
    pub egress_capacity: u32,
    pub command_batch_size: u16,
    pub snapshot_trigger: SnapshotTrigger,
    pub busy_high_watermark: u32,
    pub busy_low_watermark: u32,
}
```

Purpose: validated limits and deterministic policy.

Invariants: non-zero capacities; high watermark >= low watermark; max fills bounded.

Ownership: immutable after startup.

Serialization: canonical config fingerprint included in snapshots.

### `BookCoreCommand`

```rust
pub enum BookCoreCommand {
    Place(OrderEnvelope),
    Cancel(CancelEnvelope),
    Replace(ReplaceEnvelope),
    Admin(AdminCommand),
    Shutdown { drain: bool },
}
```

Purpose: typed inbound commands.

Invariants: fixed-width IDs and fixed-point values.

Ownership: moved from source into runtime; no borrowed payloads.

Serialization: gateway may serialize externally; core uses typed values.

### `BookCoreResponse`

```rust
pub enum BookCoreResponse {
    Accepted { command_id: CommandId, sequence: BookSequence, event: AppendedEvent },
    Rejected { command_id: CommandId, code: RejectCode, sequence: Option<BookSequence> },
    Filled { command_id: CommandId, sequence: BookSequence, event: AppendedEvent },
    PartiallyFilled { command_id: CommandId, sequence: BookSequence, event: AppendedEvent, remaining_qty: Qty },
    Cancelled { command_id: CommandId, sequence: BookSequence, event: AppendedEvent },
    Replaced { command_id: CommandId, sequence: BookSequence, event: AppendedEvent },
    EngineBusy { command_id: Option<CommandId>, retry_after_ticks: u32 },
}
```

Purpose: bounded output contract.

Invariants: success responses contain durable event identity.

Ownership: moved into response sink.

Serialization: external adapters encode it; core keeps fixed schema.

### `BookCoreState`

```rust
pub struct BookCoreState {
    pub book_id: BookId,
    pub bids: PriceSideLevels,
    pub asks: PriceSideLevels,
    pub orders: OrderIndex,
    pub arenas: BookArenas,
    pub last_event_hash: EventHash,
    pub last_applied_sequence: BookSequence,
}
```

Purpose: live book state.

Invariants: bid/ask levels canonical; orders indexed once; last hash matches durable head after append.

Ownership: mutable only by `BookCore`.

Serialization: snapshot canonical order only.

### `BookCoreMetrics`

```rust
pub struct BookCoreMetrics {
    pub commands_total: u64,
    pub accepts_total: u64,
    pub rejects_total: u64,
    pub risk_rejects_total: u64,
    pub engine_busy_total: u64,
    pub append_failures_total: u64,
    pub response_failures_total: u64,
    pub last_sequence: BookSequence,
    pub last_durable_sequence: BookSequence,
    pub ingress_depth: u32,
    pub egress_depth: u32,
}
```

Purpose: bounded observability state.

Invariants: counters monotonic except reset at process start with exported metadata.

Ownership: writer-owned; exporter reads snapshots.

Serialization: metrics export only, not replay source.

### `ProcessOrderContext`

```rust
pub struct ProcessOrderContext {
    pub command_id: CommandId,
    pub book_id: BookId,
    pub sequence: BookSequence,
    pub order_id: OrderId,
    pub account_id: AccountId,
    pub side: Side,
    pub price: Price,
    pub quantity: Qty,
    pub time_in_force: TimeInForce,
}
```

Purpose: immutable per-command processing context.

Invariants: validated fixed-point values; sequence assigned once.

Ownership: stack-owned during processing; references forbidden in durable state.

Serialization: event builder copies canonical fields.

### `PipelineResult`

```rust
pub struct PipelineResult {
    pub risk: RiskDecision,
    pub matching: MatchingResult,
    pub clearing: ClearingResult,
    pub response_kind: ResponseKind,
}
```

Purpose: stage outputs before event construction.

Invariants: clearing deltas match fills; response kind matches matching outcome.

Ownership: stack/scratch-owned; event builder consumes or borrows.

Serialization: transformed into `EngineEvent`.

### `BookSequence`

```rust
pub struct BookSequence(pub u64);
```

Purpose: book-local monotonic sequence.

Invariants: non-zero after genesis if configured; strictly increasing.

Ownership: copied by value.

Serialization: little-endian or canonical event schema as defined by events crate.

### `BookRuntimeState`

```rust
pub enum BookRuntimeState {
    Starting,
    Replaying,
    Running,
    Busy,
    Draining,
    Snapshotting,
    Quarantined,
    Stopped,
}
```

Purpose: lifecycle state.

Invariants: fatal paths cannot return to running without replay/restart.

Ownership: writer-owned, exported by copy.

Serialization: ops status only.

### `BookCoreError`

```rust
pub enum BookCoreError {
    InvalidCommand(RejectCode),
    InvalidPrice,
    InvalidQuantity,
    DuplicateCommand(CommandId),
    RiskRejected(RejectCode),
    EngineBusy,
    UnknownOrder(OrderId),
    CapacityExceeded(&'static str),
    ResponseQueueFull,
}
```

Purpose: recoverable or client-visible errors.

Invariants: stable code mapping.

Ownership: returned by value.

Serialization: stable response code.

### `BookCoreFatalError`

```rust
pub enum BookCoreFatalError {
    SequenceOverflow,
    ArithmeticOverflow,
    InvariantViolation(&'static str),
    EventAppendFailed,
    HashChainMismatch,
    ReplayDiverged,
    SnapshotInvalid,
    ConfigurationInvalid(&'static str),
}
```

Purpose: errors requiring quarantine or restart.

Invariants: never mapped to success.

Ownership: returned by value and logged.

Serialization: ops event and fatal diagnostic code.

### `SnapshotTrigger`

```rust
pub enum SnapshotTrigger {
    Disabled,
    EveryEvents(u64),
    EveryBytes(u64),
    Manual,
    Composite { events: u64, bytes: u64 },
}
```

Purpose: snapshot policy.

Invariants: zero intervals invalid unless disabled.

Ownership: immutable config.

Serialization: config fingerprint.

### `ReplayMode`

```rust
pub enum ReplayMode {
    Strict,
    FastVerifyAtEnd,
    Diagnose,
}
```

Purpose: replay verification level.

Invariants: production recovery uses `Strict` unless explicitly approved.

Ownership: startup/recovery parameter.

Serialization: ops metadata only.

### `BackpressureDecision`

```rust
pub enum BackpressureDecision {
    Accept,
    ShedNew,
    DrainOnly,
    EnterBusyMode,
    Fatal(BookCoreFatalError),
}
```

Purpose: bounded memory protection.

Invariants: `ShedNew` and `EnterBusyMode` do not mutate book state.

Ownership: computed by runtime.

Serialization: metrics/log code.

## 46. Rust-Style Pseudocode

The pseudocode below is close to implementable Rust, but it is handbook guidance rather than production source.

### `run_book_core_loop()`

```rust
fn run_book_core_loop(runtime: &mut RuntimeParts) -> Result<(), BookCoreFatalError> {
    runtime.state = BookRuntimeState::Running;
    while runtime.state == BookRuntimeState::Running || runtime.state == BookRuntimeState::Busy {
        let decision = handle_backpressure(runtime);
        match decision {
            BackpressureDecision::Accept => {
                if let Some(command) = poll_command(runtime)? {
                    match process_command(&mut runtime.core, command, &mut runtime.responses) {
                        Ok(()) => runtime.metrics.record_loop_ok(),
                        Err(error) => runtime.metrics.record_error(&error),
                    }
                } else {
                    runtime.idle_strategy.on_empty_poll();
                }
            }
            BackpressureDecision::ShedNew => enter_engine_busy_mode(runtime),
            BackpressureDecision::DrainOnly => runtime.state = BookRuntimeState::Draining,
            BackpressureDecision::EnterBusyMode => enter_engine_busy_mode(runtime),
            BackpressureDecision::Fatal(fatal) => return Err(fatal),
        }
        maybe_trigger_snapshot(runtime)?;
        if runtime.shutdown_requested {
            shutdown_book_core(runtime)?;
            break;
        }
    }
    Ok(())
}
```

### `poll_command()`

```rust
fn poll_command(runtime: &mut RuntimeParts) -> Result<Option<BookCoreCommand>, BookCoreFatalError> {
    match runtime.commands.poll_command() {
        Ok(command) => Ok(command),
        Err(BookCoreError::EngineBusy) => Ok(None),
        Err(other) => {
            runtime.metrics.record_error(&other);
            Ok(None)
        }
    }
}
```

### `process_command()`

```rust
fn process_command(
    core: &mut BookCoreParts,
    command: BookCoreCommand,
    responses: &mut dyn BookResponseSink,
) -> Result<(), BookCoreError> {
    validate_command_shape(&command)?;
    let sequence = assign_book_sequence(core).map_err(BookCoreError::from_fatal)?;
    match command {
        BookCoreCommand::Place(order) => {
            let ctx = ProcessOrderContext::from_place(core.config.book_id, sequence, order)?;
            let result = process_order(core, ctx)?;
            append_event_before_response(core, &result, responses)
        }
        BookCoreCommand::Cancel(cancel) => process_cancel_command(core, sequence, cancel, responses),
        BookCoreCommand::Replace(replace) => process_replace_command(core, sequence, replace, responses),
        BookCoreCommand::Admin(admin) => process_admin_command(core, sequence, admin, responses),
        BookCoreCommand::Shutdown { drain } => {
            core.runtime_state = if drain { BookRuntimeState::Draining } else { BookRuntimeState::Stopped };
            Ok(())
        }
    }
}
```

### `process_order()`

```rust
fn process_order(core: &mut BookCoreParts, ctx: ProcessOrderContext) -> Result<PipelineResult, BookCoreError> {
    validate_order_boundary(core.config, &ctx)?;
    let risk = invoke_risk_boundary(&mut core.risk, &ctx)?;
    let matching = invoke_matching_boundary(&mut core.matching, &mut core.state, &ctx, &risk)?;
    let clearing = invoke_clearing_boundary(&mut core.clearing, &ctx, &matching)?;
    Ok(PipelineResult { risk, matching, clearing, response_kind: ResponseKind::from_matching(&matching) })
}
```

### `validate_order_boundary()`

```rust
fn validate_order_boundary(config: &BookCoreConfig, ctx: &ProcessOrderContext) -> Result<(), BookCoreError> {
    if ctx.book_id != config.book_id { return Err(BookCoreError::InvalidCommand(RejectCode::WrongBook)); }
    if ctx.quantity.is_zero() { return Err(BookCoreError::InvalidQuantity); }
    if ctx.price.is_zero() && ctx.requires_limit_price() { return Err(BookCoreError::InvalidPrice); }
    if !ctx.price.matches_scale(config.price_scale) { return Err(BookCoreError::InvalidPrice); }
    if !ctx.quantity.matches_scale(config.quantity_scale) { return Err(BookCoreError::InvalidQuantity); }
    Ok(())
}
```

### `assign_book_sequence()`

```rust
fn assign_book_sequence(core: &mut BookCoreParts) -> Result<BookSequence, BookCoreFatalError> {
    let current = core.next_sequence;
    let next = current.checked_add_one().ok_or(BookCoreFatalError::SequenceOverflow)?;
    core.next_sequence = next;
    core.metrics.last_sequence = current;
    Ok(current)
}
```

### `invoke_risk_boundary()`

```rust
fn invoke_risk_boundary(risk: &mut dyn RiskBoundary, ctx: &ProcessOrderContext) -> Result<RiskDecision, BookCoreError> {
    let decision = risk.invoke_risk_boundary(ctx)?;
    if decision.is_reject() {
        return Err(BookCoreError::RiskRejected(decision.reject_code()));
    }
    Ok(decision)
}
```

### `invoke_matching_boundary()`

```rust
fn invoke_matching_boundary(
    matching: &mut dyn MatchingBoundary,
    state: &mut BookCoreState,
    ctx: &ProcessOrderContext,
    risk: &RiskDecision,
) -> Result<MatchingResult, BookCoreError> {
    let before_hash = state.state_hash();
    let result = matching.invoke_matching_boundary(state, ctx, risk)?;
    debug_assert!(result.open_quantity_non_negative());
    debug_assert!(state.state_hash() != before_hash || result.is_noop_allowed());
    Ok(result)
}
```

### `invoke_clearing_boundary()`

```rust
fn invoke_clearing_boundary(
    clearing: &mut dyn ClearingBoundary,
    ctx: &ProcessOrderContext,
    matching: &MatchingResult,
) -> Result<ClearingResult, BookCoreError> {
    let clearing_result = clearing.invoke_clearing_boundary(ctx, matching)?;
    if !clearing_result.conserves_quantity() { return Err(BookCoreError::from_fatal(BookCoreFatalError::InvariantViolation("clearing conservation"))); }
    Ok(clearing_result)
}
```

### `build_engine_event()`

```rust
fn build_engine_event(core: &mut BookCoreParts, ctx: &ProcessOrderContext, result: &PipelineResult) -> Result<EngineEvent, BookCoreError> {
    core.event_builder.build_engine_event(ctx, result, core.state.last_event_hash)
}
```

### `append_event_before_response()`

```rust
fn append_event_before_response(
    core: &mut BookCoreParts,
    result: &PipelineResult,
    responses: &mut dyn BookResponseSink,
) -> Result<(), BookCoreError> {
    let event = build_engine_event(core, result.context(), result)?;
    let appended = core.event_appender
        .append_event_before_response(event)
        .map_err(BookCoreError::from_fatal)?;
    core.state.last_event_hash = appended.event_hash;
    core.state.last_applied_sequence = appended.sequence;
    core.metrics.last_durable_sequence = appended.sequence;
    publish_response(responses, result, appended)
}
```

### `publish_response()`

```rust
fn publish_response(
    responses: &mut dyn BookResponseSink,
    result: &PipelineResult,
    appended: AppendedEvent,
) -> Result<(), BookCoreError> {
    let response = BookCoreResponse::from_pipeline_result(result, appended);
    responses.try_publish(response)
}
```

### `maybe_trigger_snapshot()`

```rust
fn maybe_trigger_snapshot(runtime: &mut RuntimeParts) -> Result<(), BookCoreFatalError> {
    let trigger = runtime.core.config.snapshot_trigger;
    if runtime.snapshot.should_snapshot(trigger, &runtime.core.metrics) {
        runtime.state = BookRuntimeState::Snapshotting;
        let last = runtime.core.last_appended_event();
        runtime.snapshot.write_snapshot(&runtime.core.state, last)
            .map_err(|_| BookCoreFatalError::SnapshotInvalid)?;
        runtime.state = BookRuntimeState::Running;
    }
    Ok(())
}
```

### `handle_backpressure()`

```rust
fn handle_backpressure(runtime: &RuntimeParts) -> BackpressureDecision {
    let ingress = runtime.commands.depth();
    let egress = runtime.responses.depth();
    if runtime.core.invariant_failed() { return BackpressureDecision::Fatal(BookCoreFatalError::InvariantViolation("runtime")); }
    if egress >= runtime.responses.capacity() { return BackpressureDecision::EnterBusyMode; }
    if ingress >= runtime.core.config.busy_high_watermark as usize { return BackpressureDecision::ShedNew; }
    if runtime.shutdown_requested { return BackpressureDecision::DrainOnly; }
    BackpressureDecision::Accept
}
```

### `enter_engine_busy_mode()`

```rust
fn enter_engine_busy_mode(runtime: &mut RuntimeParts) {
    runtime.state = BookRuntimeState::Busy;
    runtime.core.metrics.engine_busy_total += 1;
    let busy = BookCoreResponse::EngineBusy { command_id: None, retry_after_ticks: runtime.busy_retry_ticks };
    let _ = runtime.responses.try_publish(busy);
    if runtime.commands.depth() <= runtime.core.config.busy_low_watermark as usize {
        runtime.state = BookRuntimeState::Running;
    }
}
```

### `shutdown_book_core()`

```rust
fn shutdown_book_core(runtime: &mut RuntimeParts) -> Result<(), BookCoreFatalError> {
    runtime.state = BookRuntimeState::Draining;
    while runtime.commands.depth() > 0 && runtime.shutdown_drain {
        if let Some(command) = poll_command(runtime)? {
            process_command(&mut runtime.core, command, &mut runtime.responses)
                .map_err(BookCoreFatalError::from_recoverable_if_fatal)?;
        }
    }
    maybe_trigger_snapshot(runtime)?;
    runtime.state = BookRuntimeState::Stopped;
    Ok(())
}
```

### `replay_book_core_from_snapshot()`

```rust
fn replay_book_core_from_snapshot(
    snapshot: BookSnapshot,
    events: EventStream,
    mode: ReplayMode,
) -> Result<BookCoreState, BookCoreFatalError> {
    snapshot.verify_metadata()?;
    let mut state = BookCoreState::from_snapshot(snapshot)?;
    let mut previous_hash = state.last_event_hash;
    for event in events {
        event.verify_previous_hash(previous_hash)?;
        event.verify_canonical_hash()?;
        state.apply_replay_event(&event)?;
        previous_hash = event.event_hash();
        if mode == ReplayMode::Strict { state.verify_invariants()?; }
    }
    if mode != ReplayMode::FastVerifyAtEnd { state.verify_state_hash()?; }
    Ok(state)
}
```

### `recover_after_crash()`

```rust
fn recover_after_crash(recovery: &mut RecoveryParts) -> Result<BookCore, BookCoreFatalError> {
    let snapshot = recovery.snapshot_provider.restore_latest()?.ok_or(BookCoreFatalError::SnapshotInvalid)?;
    let event_suffix = recovery.event_log.read_suffix_after(snapshot.last_sequence)?;
    let state = replay_book_core_from_snapshot(snapshot, event_suffix, ReplayMode::Strict)?;
    let next_sequence = state.last_applied_sequence.checked_add_one().ok_or(BookCoreFatalError::SequenceOverflow)?;
    BookCore::from_recovered_state(state, next_sequence, recovery.config.clone())
}
```

## 47. Testing Strategy

Testing must prove the Book Core follows the event-sourced single-writer contract. Every test should use fixed-point integer values and deterministic seeds. Test fixtures must record command order, expected events, expected responses, and expected final state hash.

Test categories:

- unit tests for pure boundaries;
- integration tests for full pipeline;
- property tests for invariants across generated command streams;
- replay tests for crash and hash-chain cases;
- determinism tests for live/replay equivalence;
- performance and memory benchmarks for hot path regressions.

## 48. Unit Tests

| ID | Test | Expected result |
| --- | --- | --- |
| BU-001 | command parsing from typed order envelope | valid fields become `BookCoreCommand::Place` |
| BU-002 | sequence allocation monotonicity | sequences increment by one per assignment |
| BU-003 | sequence overflow | returns `BookCoreFatalError::SequenceOverflow` |
| BU-004 | invalid price rejection | no state mutation; stable reject code |
| BU-005 | invalid quantity rejection | no state mutation; stable reject code |
| BU-006 | invalid command rejection | no event append unless reject-event policy enabled |
| BU-007 | error mapping | recoverable and fatal errors map to correct response/status |
| BU-008 | backpressure decision accept | low depths return `Accept` |
| BU-009 | backpressure decision busy | high depth returns `ShedNew` or `EnterBusyMode` |
| BU-010 | config validation | zero capacity and invalid scales fail startup |
| BU-011 | event builder canonical hash | same input produces same hash |
| BU-012 | response sink full | no silent success drop |

## 49. Integration Tests

| ID | Test | Expected result |
| --- | --- | --- |
| BI-001 | accepted order | event appended, success response contains event hash |
| BI-002 | rejected order | deterministic reject response, no book mutation |
| BI-003 | full fill | maker/taker quantities close to zero open as expected |
| BI-004 | partial fill | remaining quantity rests or cancels per time-in-force |
| BI-005 | cancel | order removed and cancel event appended |
| BI-006 | replace | old priority rules and new quantity/price rules followed |
| BI-007 | risk reject | no matching invocation and stable reject response |
| BI-008 | event append | append called exactly once before success response |
| BI-009 | response publish | response references appended event identity |
| BI-010 | response queue full after append | durable event remains recoverable |

## 50. Property Tests

| ID | Property | Assertion |
| --- | --- | --- |
| BP-001 | sequence monotonicity | generated accepted commands have strictly increasing book sequences |
| BP-002 | no duplicate event hash | distinct canonical events do not reuse hash in generated stream |
| BP-003 | no response before event append | instrumented sink observes append first for every success |
| BP-004 | no negative open quantity | all nodes maintain open quantity >= zero |
| BP-005 | no state mutation without event | state hash change implies appended event or allowed reject-event |
| BP-006 | bounded active orders | active count never exceeds configured capacity |
| BP-007 | deterministic generated replay | generated stream live hash equals replay hash |

## 51. Replay Tests

| ID | Test | Expected result |
| --- | --- | --- |
| BR-001 | snapshot + event replay | final state hash equals live hash |
| BR-002 | crash after append before response | replay restores durable result; duplicate retry returns existing result |
| BR-003 | crash before append | no durable mutation; retry processes normally |
| BR-004 | duplicate retry after unknown response | stable duplicate response references existing event |
| BR-005 | hash chain verification | mismatched previous hash fails replay |
| BR-006 | incomplete snapshot | ignored in favor of previous verified snapshot |
| BR-007 | replay does not call risk | mock risk boundary call count remains zero |

## 52. Determinism Tests

| ID | Test | Expected result |
| --- | --- | --- |
| BD-001 | same commands same state | repeated run produces same state hash |
| BD-002 | same commands same events | canonical event bytes identical |
| BD-003 | map iteration stability | price-level traversal order stable across runs |
| BD-004 | no wall-clock ordering | injected clock changes do not alter matching order |
| BD-005 | fixed-point overflow | overflow outcome stable and classified |

## 53. Performance Benchmarks

| ID | Benchmark | Measurement |
| --- | --- | --- |
| BF-001 | steady-state `process_order` latency | p50/p95/p99/p999 under release build |
| BF-002 | ring drain throughput | commands per second for configured batch sizes |
| BF-003 | append boundary overhead | latency added by durable appender stub and production appender |
| BF-004 | snapshot trigger overhead | cost at safe boundary, not inner match loop |
| BF-005 | busy mode throughput | rejects per second without state mutation |

Benchmark acceptance must be defined per deployment target and kept in CI as regression thresholds where practical.

## 54. Memory Benchmarks

| ID | Benchmark | Measurement |
| --- | --- | --- |
| BM-001 | zero allocation assertion | no allocations after warmup during steady-state matching |
| BM-002 | arena capacity | max active orders uses bounded memory |
| BM-003 | pool recycle | repeated fills/cancels do not grow memory |
| BM-004 | event buffer capacity | max fills fit configured event scratch |
| BM-005 | snapshot buffer | bounded snapshot memory under max book state |

## 55. Unsafe Code Policy

Initial implementation must use no `unsafe` code.

Any future unsafe proposal requires:

- isolated module;
- safe fallback;
- proof comment explaining aliasing, lifetime, and alignment assumptions;
- Miri tests;
- fuzz/property tests covering boundary conditions;
- benchmark showing material benefit;
- reviewer sign-off.

Unsafe must never be used to bypass single-writer ownership or create mutable aliases to book state.

## 56. Dependency Policy

Allowed production dependencies are narrow domain crates and audited low-level utilities.

Allowed:

- `hermes-fixed` for fixed-point integer types;
- `hermes-ids` for fixed-width identifiers;
- `hermes-events` for event schemas and hash types;
- trait-only surfaces from matching, risk, clearing, snapshot, and replay crates;
- `core`/`std` primitives where appropriate.

Forbidden in hot path:

- SQL drivers;
- Kafka clients;
- HTTP servers/clients;
- async executors;
- float math crates;
- random generators for sequencing;
- unbounded logging/telemetry exporters;
- dynamic scripting engines.

## 57. Implementation Sequence

1. Define public types, traits, and config validation.
2. Implement `BookSequence` and sequence allocator tests.
3. Implement bounded command/response contracts and error mapping.
4. Implement memory arenas and pools with capacity tests.
5. Implement `BookCoreState` with deterministic state hash.
6. Implement validation boundary.
7. Implement risk, matching, and clearing trait adapters using mocks.
8. Implement event builder and append-before-response orchestration.
9. Implement runtime loop and backpressure.
10. Implement snapshot and replay boundaries.
11. Add integration tests for accepted/rejected/fill/cancel/replace flows.
12. Add property and replay tests.
13. Add performance and memory benchmarks.
14. Run dependency and unsafe audits.

## 58. Codex Implementation Tasks

Task set for future agents:

- Create `crates/hermes-book` skeleton only when explicitly requested.
- Add `Cargo.toml` with allowed dependencies and no forbidden hot-path crates.
- Implement `config.rs`, `error.rs`, `sequence.rs`, `command.rs`, and `response.rs` first.
- Add unit tests BU-001 through BU-012.
- Implement memory modules with preallocation and no steady-state growth.
- Implement state store with deterministic price-level order.
- Implement pipeline modules with trait boundaries and mocks.
- Implement event append-before-response test harness.
- Implement runtime loop after pure processing tests pass.
- Implement replay only after event schema golden tests pass.
- Add property tests BP-001 through BP-007.
- Add benchmarks BF-001 through BF-005 and BM-001 through BM-005.
- Do not create gateway, risk, clearing, matching, DB, Kafka, or cloud adapters in this crate.

## 59. Review Checklist

Reviewers must confirm:

- [ ] exactly one writer mutates a book;
- [ ] sequence is book-local and monotonic;
- [ ] fixed-point integers only;
- [ ] no global sequencer;
- [ ] no DB/Kafka/cloud dependency in hot path;
- [ ] no heap allocation during steady-state matching;
- [ ] no locks inside `process_order`;
- [ ] event append occurs before success response;
- [ ] all success responses contain durable event identity;
- [ ] replay verifies hash chain;
- [ ] state hash live equals replay hash;
- [ ] bounded queues and bounded memory are enforced;
- [ ] failure paths fail closed;
- [ ] no production source files were created by handbook-only work;
- [ ] tests cover required matrix IDs;
- [ ] unsafe code is absent or approved under policy.

## 60. Completion Criteria

The Book Core implementation material is complete when:

- this chapter defines the concrete crate and file layout;
- every module has responsibility, public items, forbidden behaviour, dependency rules, and tests;
- public traits include purpose, method signatures, ownership, errors, determinism, and tests;
- core structs/enums include fields, purpose, invariants, ownership, and serialization rules;
- pseudocode covers runtime, command polling, processing, boundaries, append, response, snapshots, backpressure, shutdown, replay, and crash recovery;
- testing matrix includes unit, integration, property, replay, determinism, performance, and memory coverage;
- module contract and example are consistent with this chapter;
- HES remains unchanged;
- no production Rust source files, DOCX, or PDF artifacts are created.
