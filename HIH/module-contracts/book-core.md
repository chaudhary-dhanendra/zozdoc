# Module Contract: Book Core (`hermes-book`)

## Responsibility

`hermes-book` owns the single-writer Book Core for one book partition. It coordinates typed commands, book-local sequence assignment, validation, risk invocation, matching invocation, clearing delta derivation, immutable `EngineEvent` construction, hash-chained event append, response publication, snapshot boundaries, replay boundaries, and bounded observability.

The module owns all live mutable book state. Other modules interact through explicit traits and bounded queues.

## Inputs

- `BookCoreCommand::Place(OrderEnvelope)` with fixed-width IDs and fixed-point price/quantity.
- `BookCoreCommand::Cancel(CancelEnvelope)`.
- `BookCoreCommand::Replace(ReplaceEnvelope)`.
- `BookCoreCommand::Admin(AdminCommand)` for safe operational controls.
- `BookCoreCommand::Shutdown { drain }`.
- Verified snapshot plus event suffix during recovery.
- Config validated before startup.

Input constraints:

- no floats;
- no unbounded strings;
- no wall-clock ordering fields;
- no gateway protocol parsing in hot path;
- no command accepted from unbounded queue.

## Outputs

- `BookCoreResponse::Accepted`, `Filled`, `PartiallyFilled`, `Cancelled`, or `Replaced` after durable event append.
- `BookCoreResponse::Rejected` for deterministic non-mutating rejection or configured reject-event policy.
- `BookCoreResponse::EngineBusy` when bounded backpressure rejects before mutation.
- Immutable hash-chained `EngineEvent` records.
- Bounded metrics snapshots.
- Canonical snapshots at safe boundaries.

Success outputs must include durable event identity: book sequence, event hash, and append identity.

## Public Traits

- `BookCoreRuntime`: runs one single-writer loop and handles lifecycle states.
- `BookCommandSource`: bounded FIFO typed command source.
- `BookResponseSink`: bounded response publication sink.
- `BookCoreTrait` or `BookCoreApi`: test/runtime processing interface for `BookCore`.
- `OrderProcessor`: place/cancel/replace processing inside writer.
- `BookStateStore`: deterministic state operations and state hash.
- `SequenceAllocator`: book-local monotonic sequence assignment.
- `RiskBoundary`: deterministic risk reservation/reject boundary.
- `MatchingBoundary`: deterministic matching over writer-owned state.
- `ClearingBoundary`: deterministic clearing delta derivation.
- `EngineEventBuilder`: canonical event construction and hashing.
- `EventAppender`: append-before-response durable event boundary.
- `SnapshotProvider`: safe snapshot trigger/write/restore interface.
- `ReplayEngine`: verified snapshot-plus-event replay.
- `MetricsRecorder`: bounded metric recording with no hot-path allocation.

Each trait must define ownership, error behaviour, determinism requirements, and tests as specified in `HIH-003 Book Core Implementation`.

## Structs and Enums

Required public or crate-public contracts:

- `BookCore`;
- `BookCoreConfig`;
- `BookCoreCommand`;
- `BookCoreResponse`;
- `BookCoreState`;
- `BookCoreMetrics`;
- `ProcessOrderContext`;
- `PipelineResult`;
- `BookSequence`;
- `BookRuntimeState`;
- `BookCoreError`;
- `BookCoreFatalError`;
- `SnapshotTrigger`;
- `ReplayMode`;
- `BackpressureDecision`.

Serialization rules:

- live `BookCore` is not serialized;
- commands and responses use stable external schemas outside the hot implementation;
- snapshots serialize canonical state order, last sequence, last event hash, config fingerprint, and state hash;
- events use canonical event serialization from the events module.

## Forbidden Behaviour

The Book Core must not:

- expand or reinterpret HES;
- use a global sequencer;
- use floats;
- allocate heap memory during steady-state matching;
- lock inside `process_order`;
- publish success before event append;
- mutate state from outside `BookCore`;
- call SQL, Kafka, cloud, HTTP, or external network services in the hot path;
- use system time, randomness, or non-deterministic iteration for ordering;
- create unbounded queues, caches, buffers, logs, or maps;
- call live risk service during replay;
- silently drop responses;
- convert append failure into success;
- create production Rust source files from handbook-only work.

## Dependencies Allowed

Production hot-path dependencies:

- `hermes-fixed` for fixed-point integer values;
- `hermes-ids` for fixed-width identifiers;
- `hermes-events` for event schemas, hashes, and append interfaces;
- trait surfaces from matching, risk, clearing, replay, and snapshot modules;
- audited bounded collections or locally implemented arenas/pools;
- standard library primitives that do not violate hot-path constraints.

Development/test dependencies:

- property testing framework with deterministic seeds;
- benchmark harness;
- allocation-counting test helper;
- mock boundary implementations.

## Dependencies Forbidden

Forbidden in hot modules and transitive hot dependency graph:

- SQL/database drivers;
- Kafka clients;
- HTTP servers or clients;
- cloud SDKs;
- async executors for matching path;
- float math crates;
- random-number generators for sequence/order priority;
- dynamic scripting engines;
- unbounded telemetry exporters;
- JSON parsing as command ingress inside Book Core.

## Invariants

- Exactly one active writer mutates a book.
- Book sequences are local to one book and strictly monotonic.
- Every durable state mutation is represented by an immutable event.
- Success response is externally visible only after event append.
- Event log is hash chained and verified during replay.
- Replay from snapshot plus event suffix produces the same state hash as live processing.
- All price and quantity arithmetic uses fixed-point integers.
- Queue and memory capacity are bounded and validated at startup.
- `ENGINE_BUSY` paths do not mutate book state.
- Risk reject occurs before matching mutation.
- Clearing deltas conserve quantities and preserve fill order.
- Snapshot metadata includes last sequence and last event hash.
- Fatal append, hash, sequence, arithmetic, and invariant failures quarantine the writer.

## Test Requirements

Unit tests:

- BU-001 command parsing from typed envelope;
- BU-002 sequence allocation monotonicity;
- BU-003 sequence overflow;
- BU-004 invalid price rejection;
- BU-005 invalid quantity rejection;
- BU-006 invalid command rejection;
- BU-007 error mapping;
- BU-008 backpressure accept;
- BU-009 backpressure busy;
- BU-010 config validation;
- BU-011 canonical event hash;
- BU-012 response sink full.

Integration tests:

- BI-001 accepted order;
- BI-002 rejected order;
- BI-003 full fill;
- BI-004 partial fill;
- BI-005 cancel;
- BI-006 replace;
- BI-007 risk reject;
- BI-008 event append before response;
- BI-009 response publish with event identity;
- BI-010 response queue full after append.

Property tests:

- BP-001 sequence monotonicity;
- BP-002 no duplicate event hash;
- BP-003 no response before event append;
- BP-004 no negative open quantity;
- BP-005 no state mutation without event;
- BP-006 bounded active orders;
- BP-007 deterministic generated replay.

Replay tests:

- BR-001 snapshot plus event replay;
- BR-002 crash after append before response;
- BR-003 crash before append;
- BR-004 duplicate retry after unknown response;
- BR-005 hash chain verification;
- BR-006 incomplete snapshot ignored;
- BR-007 replay does not call risk.

Determinism tests:

- BD-001 same commands same state;
- BD-002 same commands same events;
- BD-003 stable price-level traversal;
- BD-004 no wall-clock ordering;
- BD-005 fixed-point overflow classification.

## Benchmark Requirements

Performance benchmarks:

- BF-001 steady-state `process_order` latency;
- BF-002 ring drain throughput;
- BF-003 append boundary overhead;
- BF-004 snapshot trigger overhead;
- BF-005 busy mode throughput.

Memory benchmarks:

- BM-001 zero allocation after warmup;
- BM-002 bounded arena capacity;
- BM-003 pool recycle without growth;
- BM-004 event buffer capacity for max fills;
- BM-005 snapshot buffer bounded memory.

Benchmarks must run in release mode and must fail or warn on allocation regressions in steady-state matching.

## Related HES References

- Single-writer Book Core.
- Book-local ordering.
- Fixed-point arithmetic.
- Event sourcing and immutable `EngineEvent` records.
- Hash-chained engine events.
- Deterministic replay.
- No database in the hot path.
- No Kafka before decision.
- Bounded lock-free rings.
- Hot/cold path separation.
- Production readiness gates.

## Completion Checklist

- [ ] `hermes-book` crate layout matches HIH-003.
- [ ] Public traits are implemented with deterministic mocks.
- [ ] Required structs/enums exist with stable invariants.
- [ ] Config validation rejects unsafe bounds.
- [ ] Sequence allocation is book-local and monotonic.
- [ ] Risk boundary occurs before matching mutation.
- [ ] Matching performs no steady-state heap allocation.
- [ ] Clearing deltas are fixed-point and conservative.
- [ ] EngineEvent build is canonical and immutable.
- [ ] Event append occurs before every success response.
- [ ] Response queue full behaviour is explicit.
- [ ] Snapshot metadata includes sequence and event hash.
- [ ] Replay verifies snapshot, hash chain, and state hash.
- [ ] Backpressure returns `ENGINE_BUSY` before mutation.
- [ ] Unit, integration, property, replay, determinism, performance, and memory tests are present.
- [ ] Dependency audit shows no forbidden hot-path dependencies.
- [ ] Unsafe code is absent or approved under policy.
