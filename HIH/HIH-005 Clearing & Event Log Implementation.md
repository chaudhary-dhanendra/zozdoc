# HIH-005 Clearing & Event Log Implementation

## 1. Purpose

This chapter defines implementation-grade guidance for `hermes-clearing` and `hermes-events`, plus their integration boundaries with `hermes-ledger` and `hermes-replay`. It is intentionally prescriptive about module ownership, trait contracts, event lifecycle, serialization, hash chaining, recovery, testing, and benchmarks so engineers can implement the crates without inventing architecture.

HIH-005 does not restate HES as protocol law. It translates the frozen HermesNet principles into Rust implementation decisions:

- every accepted engine decision is represented by an immutable `EngineEvent`;
- the event log is append-only and hash-chained;
- externally visible success is published only after the event append is durable under the configured fsync policy;
- replay from genesis or from a verified snapshot reconstructs identical state;
- clearing and accounting use fixed-point integer arithmetic only;
- historical events are never edited, compacted in place, or reversed except by compensating events.

## 2. Related HES Volumes

Implementation must remain consistent with the HES volumes covering matching, risk, clearing, settlement, event logging, replay, observability, security, and operations. HIH-005 depends on these HES requirements:

- single-writer Book Core with book-local ordering;
- no database dependency in the hot path;
- no Kafka before the matching/clearing decision is committed;
- deterministic identifiers, timestamps, ordering, and serialization inputs;
- auditable financial mutations;
- bounded queues and bounded recovery work;
- fixed-point arithmetic for notional, quantity, fee, ledger, and wallet values.

When this chapter appears more specific than HES, treat it as an implementation choice that must still satisfy HES. When HES is stricter, HES wins.

## 3. Scope

HIH-005 covers:

- `crates/hermes-clearing/` pure clearing, fee, wallet delta, ledger delta, and settlement journal construction;
- `crates/hermes-events/` event construction, canonical binary serialization, append-only event log, hash chain, schema/version handling, and replay event source adapters;
- the ledger integration boundary: clearing emits deterministic ledger deltas and journals but does not mutate ledger storage directly;
- the replay integration boundary: events expose verified streams and snapshot-aware streams but replay state machines live outside the event log writer;
- crash recovery, corruption detection, retention, archival, observability, performance, testing, and review criteria.

## 4. Non-goals

- Do not generate production Rust source files from this chapter.
- Do not modify HES semantics.
- Do not create DOCX or PDF artifacts.
- Do not introduce SQL, Kafka, object storage, HTTP, random number generation, or wall-clock reads into deterministic hot-path modules.
- Do not define ledger persistence internals; this chapter defines only the event-to-ledger boundary.
- Do not define replay projection internals; this chapter defines only verified event sources and replay expectations.

## 5. Clearing Architecture

`hermes-clearing` is a deterministic library crate. It converts Book Core execution facts into accounting-ready artifacts:

1. validated execution inputs enter `ClearingEngine`;
2. `FeeCalculator` computes maker/taker fees and rebates with fixed-point rounding rules;
3. `WalletDeltaBuilder` derives available/held wallet movements;
4. `LedgerDeltaBuilder` derives double-entry ledger movements;
5. `JournalBuilder` groups deltas into a settlement journal;
6. `EngineEventBuilder` receives the clearing result and creates an immutable event payload for `hermes-events`.

The crate must be pure for a given input. It must not append events itself, read storage, read clocks, publish messages, or call ledger services. Its output is data that can be serialized into an `EngineEvent` and replayed.

## 6. Clearing Responsibilities

`hermes-clearing` owns:

- fee lookup and calculation for maker/taker roles;
- fee currency selection and fixed-point rounding;
- wallet delta construction for available, held, debit, credit, fee, and rebate movements;
- ledger delta construction for customer, fee revenue, rebate, settlement, and clearing suspense accounts;
- settlement journal construction with debit/credit balance validation;
- deterministic clearing result identifiers derived from input event context;
- clearing-specific errors and invariant checks.

It does not own:

- order matching;
- risk reservation approval;
- durable event append;
- ledger storage mutation;
- replay projection storage;
- market data publication.

## 7. EngineEvent Architecture

`hermes-events` owns the immutable envelope for every engine decision. An `EngineEvent` has:

- `EngineEventHeader`: version, kind, book, sequence, command id, causation id, correlation id, deterministic engine timestamp, previous hash, payload hash, and current hash;
- `EngineEventPayload`: accepted/rejected order, trade, cancel, replace, risk reservation, clearing result, settlement journal, compensating adjustment, snapshot marker, or operational marker;
- canonical binary bytes used for hashing and durable append;
- append metadata stored outside the hash input, such as segment path and byte offset.

The event header and payload are immutable after hash calculation. If a correction is required, append a new compensating event that references the original event.

## 8. Event Lifecycle

1. Book Core produces a deterministic decision with book-local sequence.
2. Clearing builds deltas and journals for accepted executions or rejection facts for rejected commands.
3. `EngineEventBuilder` constructs a fully populated event with `prev_hash` from the current hash-chain state.
4. `EngineEventSerializer` canonicalizes the event into bytes.
5. `HashChain` computes `payload_hash` and `event_hash` from canonical bytes and the previous hash.
6. `EventAppender` writes length-prefixed record bytes to the active segment.
7. Appender writes record checksum/trailer and advances only after the record is complete.
8. Appender executes the configured durability policy.
9. The in-memory hash-chain head and durable sequence advance.
10. Only then may the caller report externally visible success.

## 9. Event Append Pipeline

The append pipeline is single-writer per book stream unless a higher-level sequencer proves equivalent book-local ordering. The hot path is:

```text
ClearingResult -> EngineEvent -> canonical bytes -> checksum -> hash-chain link -> segment append -> durability fence -> success publication
```

Rules:

- append is synchronous from the perspective of the engine decision;
- partial writes are not committed;
- segment metadata must never be part of `event_hash`;
- every record includes enough framing to skip, verify, or truncate during recovery;
- appender must reject non-monotonic book sequence numbers;
- appender must reject `prev_hash` that does not equal the current chain head.

## 10. Clearing Pipeline

Accepted executions follow this deterministic pipeline:

1. normalize execution input into `ClearingInput`;
2. validate product, side, role, scale, and fee schedule version;
3. calculate notional using fixed-point checked arithmetic;
4. calculate maker/taker fees and rebates;
5. build wallet deltas for buyer, seller, and fee accounts;
6. build ledger deltas with debit/credit sides;
7. build settlement journal and verify balance conservation;
8. build `ClearingResult`;
9. embed the result in an `EngineEventPayload`;
10. append the event before returning success.

Rejected orders still produce auditable events when the decision reaches the event boundary. Rejections do not contain clearing deltas unless a reservation release or compensating movement is required.

## 11. Fee Calculation Implementation

Fee calculation is pure and fixed-point:

- input amount is quantity, price, notional, liquidity role, account fee tier, product id, and fee schedule version;
- rates are signed fixed-point integers so rebates can be represented without a special floating type;
- multiplication must use widened checked integer arithmetic;
- rounding mode is explicit in configuration and must be deterministic per market;
- fee output includes currency, payer account, receiver account, rate, raw amount before rounding, rounded amount, and rounding residue if retained for audit.

Fee calculation must fail closed on overflow, unsupported scale, missing fee schedule, or invalid role.

## 12. Maker/Taker Fee Model

Each fill has two sides:

- maker side: resting liquidity provider, normally lower fee or rebate;
- taker side: aggressor, normally higher fee.

`FeeCalculator` must receive the role from matching facts, not infer it from time or order id. The fee schedule lookup key should include product, account, role, schedule version, and optional volume tier snapshot already determined outside the hot calculation. If the maker receives a rebate, represent it as a ledger credit to the maker and a debit to the rebate expense or fee revenue contra account.

## 13. Wallet Delta Generation

Wallet deltas represent user-facing balance movement intent. They are not durable ledger postings by themselves.

Fields include account id, wallet id, asset id, available delta, held delta, reason, source event id, and deterministic sequence. Invariants:

- no floating values;
- available and held changes are explicit, not inferred;
- a wallet delta never mutates previous deltas;
- holds released after fill/cancel must reference the reservation or event being released;
- sum checks are performed per asset where conservation applies.

## 14. Ledger Delta Generation

Ledger deltas represent accounting postings suitable for `hermes-ledger` ingestion. They are double-entry facts:

- each delta has account, asset, debit amount, credit amount, posting type, and source event id;
- exactly one of debit or credit is non-zero per line;
- journal totals balance per asset and settlement domain;
- fees and rebates post to configured revenue/expense accounts;
- customer movements and clearing house movements are separated.

`hermes-clearing` builds ledger deltas; `hermes-ledger` validates and persists them after consuming committed events.

## 15. Settlement Journal Generation

`SettlementJournal` groups wallet and ledger deltas for one engine decision or one deterministic clearing batch. It contains journal id, source event id, book id, sequence, journal lines, wallet deltas, ledger deltas, balance proof, and status. The journal is generated before event append and becomes immutable inside the event payload.

A journal is valid only when:

- debit and credit totals balance per asset;
- every wallet delta has a corresponding ledger interpretation or documented non-ledger reason;
- fee lines reference the fee schedule version;
- the journal can be replayed without external lookup except static schema/compatibility code.

## 16. Binary Serialization Strategy

Use a canonical binary format owned by `hermes-events`:

- field order is schema-defined and stable;
- integers are fixed width and little-endian unless the schema states otherwise;
- variable-length arrays are length-prefixed with bounded lengths;
- maps are forbidden in canonical payloads unless converted to sorted vectors;
- optional fields are encoded with explicit presence tags;
- unknown fields are allowed only in extension regions defined by schema versioning;
- serialization must not include memory addresses, allocation order, segment offsets, or wall-clock values.

Golden vectors must lock bytes for representative events.

## 17. Event Versioning

`EngineEventVersion` contains major, minor, and patch-compatible schema identifiers. Rules:

- major changes may require migration/replay adapter code;
- minor changes may add optional fields with deterministic defaults;
- patch changes must not change canonical bytes for the same logical event;
- every event stores its exact version;
- serializers must refuse to emit unsupported versions;
- replay must report unsupported versions before applying state changes.

## 18. Schema Evolution

Schema evolution is append-only:

- never rename fields without a compatibility alias in decoding docs;
- never change numeric scale for an existing field;
- never reuse enum discriminants;
- reserve removed fields and discriminants;
- add new payload variants behind feature and replay compatibility gates;
- provide migration tests from old golden vectors.

## 19. Backward Compatibility

`hermes-events` must support reading all event versions required by the deployment's retention and replay policy. Compatibility code maps old payloads into the current replay model without changing historical hashes. The original canonical bytes remain the source of truth for hash verification; decoded compatibility structs are derived views.

## 20. Hash-chain Implementation

`HashChain` maintains the current head per book stream or global stream, depending on deployment topology. Hash input is:

```text
domain_separator || event_version || book_id || sequence || prev_hash || payload_hash || canonical_header_without_current_hash || canonical_payload
```

Rules:

- `prev_hash` for genesis is a configured zero or genesis hash constant;
- `event_hash` is computed after payload canonicalization;
- hash algorithm and domain separator are versioned;
- the current hash is stored in the event header and in segment index metadata;
- chain state advances only after append success.

## 21. Integrity Verification

Integrity verification checks:

- record framing length;
- record checksum;
- canonical decode;
- payload hash;
- event hash;
- previous hash linkage;
- monotonic book sequence;
- schema compatibility;
- optional segment index consistency.

Verification stops at the first invalid record and returns the last verified offset and hash-chain state.

## 22. Replay Interaction

`ReplayEventSource` exposes verified events in deterministic order. It must not yield an event until framing, checksum, schema, and hash-chain checks pass. Replay applies events to projections and compares final state with expected checkpoints when available.

Replay consumers must treat `EngineEvent` as authoritative. They must not recalculate fees using a current fee schedule; the fee amounts inside the historical event are replayed exactly.

## 23. Snapshot Interaction

`SnapshotAwareEventSource` starts from a verified snapshot containing book id, last sequence, last event hash, schema version, and projection digest. It then streams events where `prev_hash` equals the snapshot hash-chain head and sequence is greater than the snapshot sequence.

Snapshots are accelerators, not replacements for event history. A snapshot is invalid if it references an unknown event hash, unsupported schema, or projection digest mismatch.

## 24. Recovery After Crash

Recovery procedure:

1. open last sealed segment and active segment read-only;
2. verify records from the last trusted checkpoint;
3. identify the last complete valid record;
4. truncate bytes after the last valid record in the active segment;
5. rebuild hash-chain state and durable sequence;
6. replay from last valid snapshot plus suffix;
7. resume appends with `prev_hash` equal to the recovered chain head.

If corruption occurs before the last trusted checkpoint, quarantine the stream and require operator recovery from replica/archive.

## 25. Event Corruption Detection

Corruption is any mismatch in length, checksum, decode, hash, linkage, sequence, or compatibility. Detection returns `CorruptionReport` with segment id, byte offset, expected hash/checksum, observed value, last valid sequence, last valid hash, and recommended action. Detection must not attempt silent repair except truncating an incomplete active-segment tail during crash recovery.

## 26. Event Retention

Retention policy must preserve all events required for audit, replay, dispute resolution, and regulatory requirements. Hot segments may be sealed and moved to warm storage only after checksum and hash-chain verification. Retention metadata must include first/last sequence, first/last hash, schema versions, event count, and byte size.

## 27. Event Archival

Archival is cold-path and outside the append decision. Archive writers consume sealed verified segments. Archived segments remain immutable and must retain original bytes. Restore tests must prove archived bytes reproduce the same hash chain and replay state.

## 28. Configuration

Required configuration:

- event segment size and directory;
- fsync policy (`Always`, `EveryNEvents`, `EveryDuration`, or deployment-approved equivalent);
- maximum record size;
- maximum payload vector lengths;
- hash algorithm and domain separator;
- supported event versions;
- fee schedule versions and rounding modes;
- settlement accounts;
- snapshot interval and checkpoint validation policy;
- metrics namespace and log redaction policy.

Configuration is validated at startup and encoded into operational marker events when needed for audit.

## 29. Metrics

Expose counters, gauges, and histograms for:

- events appended, bytes appended, append failures;
- durable sequence and hash-chain head label/hash prefix;
- append latency, serialization latency, fsync latency;
- clearing results by kind;
- fee calculation failures by code;
- wallet delta and ledger delta counts;
- replay events/sec and replay lag;
- corruption detections;
- segment rotations, archive lag, snapshot age.

Metrics must not expose secrets, full account identifiers, or personally identifiable data.

## 30. Logging

Use structured logs with event id, book id, sequence, command id, correlation id, error code, segment id, and offset. Logs must not be the audit source of truth. Sensitive account data must be redacted or tokenized.

## 31. Tracing

Trace spans should cover clearing, event build, serialization, append, fsync, and success publication. Span ids may be stored as non-hash metadata but must not affect deterministic event bytes.

## 32. Error Handling

Errors are typed and deterministic. Key variants:

- `ClearingError::ArithmeticOverflow`;
- `ClearingError::InvalidFeeSchedule`;
- `ClearingError::UnbalancedJournal`;
- `ClearingError::InvariantViolation`;
- `EventError::SerializationFailed`;
- `EventError::AppendFailed`;
- `EventError::HashChainMismatch`;
- `EventError::UnsupportedVersion`;
- `EventError::CorruptionDetected`;
- `EventError::PartialAppendRecovered`.

After a local clearing result is computed but append fails, the engine must not publish success. It must stop, quarantine, or retry according to the durability policy without creating an ambiguous external state.

## 33. Performance Considerations

- preallocate bounded buffers for serialization;
- avoid heap allocation in per-fill fee calculation where practical;
- batch fsync only when HES/deployment policy allows delayed durability semantics;
- keep hot-path dependencies small and synchronous;
- avoid map iteration in canonical serialization;
- use segment indexes for recovery without trusting them over record verification;
- benchmark p99 and p999 latency, not only throughput.

## 34. Crate Layout and Module Responsibilities

### 34.1 `crates/hermes-clearing/`

| Module | Purpose | Ownership | Allowed dependencies | Forbidden dependencies | Public APIs | Testing responsibility |
|---|---|---|---|---|---|---|
| `engine.rs` | Orchestrates clearing pipeline. | Owns `ClearingEngine` implementation and no durable state. | fixed types, ids, local traits, metrics trait. | DB, Kafka, filesystem, clocks, networking. | `ClearingEngine::clear_execution`, `clear_rejection`. | end-to-end clearing result tests. |
| `clearing.rs` | Domain clearing deltas and invariants. | Owns `ClearingDelta`, `ClearingResult`. | fixed types, ids. | IO, async runtime. | delta constructors and validators. | spot/futures delta tests. |
| `fee.rs` | Maker/taker fees and rebates. | Owns fee schedules and calculation. | fixed math, config. | floats, wall clock. | `FeeCalculator`. | tier, rounding, overflow tests. |
| `settlement.rs` | Settlement grouping. | Owns settlement rules. | ledger delta types, wallet delta types. | ledger storage. | `SettlementBuilder`. | balance conservation tests. |
| `journal.rs` | Journal lines and proofs. | Owns journal construction. | fixed types, ids. | DB, filesystem. | `JournalBuilder`. | double-entry tests. |
| `config.rs` | Validated clearing config. | Owns config structs. | serde in cold path if selected. | hot-path parsing. | `ClearingConfig::validate`. | invalid config tests. |
| `error.rs` | Clearing error taxonomy. | Owns stable error codes. | core/std error. | secret-bearing debug output. | `ClearingError`. | error mapping tests. |
| `metrics.rs` | Metrics abstraction. | Owns recorder trait usage. | metrics trait only. | global mutable metrics in pure logic. | `EventMetricsRecorder` adapter use. | no-op and counting recorder tests. |

### 34.2 `crates/hermes-events/`

| Module | Purpose | Ownership | Allowed dependencies | Forbidden dependencies | Public APIs | Testing responsibility |
|---|---|---|---|---|---|---|
| `engine.rs` | Event builder orchestration. | Owns event construction flow. | ids, fixed types, schema, hash chain. | ledger storage, matching mutation. | `EngineEventBuilder`. | event invariant tests. |
| `serializer.rs` | Canonical binary encode/decode. | Owns byte format. | bytes, fixed types, schema. | non-deterministic serializers. | `EngineEventSerializer`. | golden vectors. |
| `hash_chain.rs` | Hash computation and verification. | Owns chain state. | approved hash crate. | random salts, wall clocks. | `HashChain`. | stable hash tests. |
| `replay.rs` | Verified event sources. | Owns stream adapters. | event log reader, snapshot metadata. | projection mutation ownership. | `ReplayEventSource`, `SnapshotAwareEventSource`. | genesis/snapshot replay tests. |
| `schema.rs` | Field and payload schema. | Owns discriminants and bounds. | ids, fixed types. | runtime network schema fetch. | schema constants. | compatibility tests. |
| `versioning.rs` | Version compatibility rules. | Owns support matrix. | schema. | implicit upgrades. | `EngineEventVersion`. | old vector decode tests. |
| `config.rs` | Event log config. | Owns validated config. | cold-path parsing. | hot-path env reads. | `EventLogConfig::validate`. | invalid config tests. |
| `error.rs` | Event error taxonomy. | Owns stable event errors. | std error. | secret leakage. | `EventError`. | error classification tests. |
| `metrics.rs` | Event metrics abstraction. | Owns event metrics trait. | metrics facade if approved. | unbounded labels. | `EventMetricsRecorder`. | label/redaction tests. |

## 35. Trait Contracts

The signatures below are Rust-style contracts, not production source.

### 35.1 `ClearingEngine`

Responsibility: deterministic conversion of execution facts into clearing results.

```rust
trait ClearingEngine {
    fn clear_execution(&mut self, input: ClearingInput<'_>) -> Result<ClearingResult, ClearingError>;
    fn clear_rejection(&mut self, input: RejectionInput<'_>) -> Result<ClearingResult, ClearingError>;
}
```

Ownership: owns transient calculators/builders or borrows them explicitly. Determinism: same input/config yields byte-equivalent `ClearingResult`. Failure: no partial mutable state escapes. Tests: accepted/rejected order, fill, cancel, replace, overflow.

### 35.2 `SettlementBuilder`

```rust
trait SettlementBuilder {
    fn build_settlement(&self, delta: &ClearingDelta) -> Result<SettlementJournal, ClearingError>;
}
```

Builds balanced journals from clearing deltas. Must reject unbalanced or unsupported assets.

### 35.3 `FeeCalculator`

```rust
trait FeeCalculator {
    fn calculate_fees(&self, fill: &FillFact, schedule: FeeScheduleRef) -> Result<FeeBreakdown, ClearingError>;
}
```

Must use checked fixed-point arithmetic and explicit rounding.

### 35.4 `WalletDeltaBuilder`

```rust
trait WalletDeltaBuilder {
    fn build_wallet_delta(&self, delta: &ClearingDelta, fees: &FeeBreakdown) -> Result<Vec<WalletDelta>, ClearingError>;
}
```

Produces bounded vectors and explicit available/held movements.

### 35.5 `LedgerDeltaBuilder`

```rust
trait LedgerDeltaBuilder {
    fn build_ledger_delta(&self, delta: &ClearingDelta, fees: &FeeBreakdown) -> Result<Vec<LedgerDelta>, ClearingError>;
}
```

Produces double-entry lines and validates per-asset balance.

### 35.6 `JournalBuilder`

```rust
trait JournalBuilder {
    fn build_settlement_journal(
        &self,
        wallet: Vec<WalletDelta>,
        ledger: Vec<LedgerDelta>,
    ) -> Result<SettlementJournal, ClearingError>;
}
```

Owns grouping, balance proof, and deterministic journal id derivation.

### 35.7 `EngineEventBuilder`

```rust
trait EngineEventBuilder {
    fn build_engine_event(&self, input: EngineEventBuildInput<'_>) -> Result<EngineEvent, EventError>;
}
```

Requires caller-supplied book sequence and previous hash. Must reject missing causation, unsupported payload version, or inconsistent hash fields.

### 35.8 `EngineEventSerializer`

```rust
trait EngineEventSerializer {
    fn serialize_event(&self, event: &EngineEvent, out: &mut BoundedBytes) -> Result<(), EventError>;
    fn deserialize_event(&self, bytes: &[u8]) -> Result<EngineEvent, EventError>;
}
```

Serialization is canonical and deterministic. Deserialization validates bounds before allocation.

### 35.9 `EventLog`

```rust
trait EventLog {
    fn append_record(&mut self, record: EventLogRecord<'_>) -> Result<EventLogPosition, EventError>;
    fn read_from(&self, position: EventLogPosition) -> Result<EventLogIterator<'_>, EventError>;
    fn truncate_to(&mut self, position: EventLogPosition) -> Result<(), EventError>;
}
```

Owns segment IO, not event construction. Must never expose partial records as committed.

### 35.10 `EventAppender`

```rust
trait EventAppender {
    fn append_event(&mut self, event: EngineEvent) -> Result<AppendedEvent, EventError>;
}
```

Coordinates serialization, hash-chain validation, segment append, durability fence, metrics, and chain-head advancement.

### 35.11 `HashChain`

```rust
trait HashChain {
    fn current_state(&self) -> HashChainState;
    fn compute_event_hash(&self, event: &EngineEvent, bytes: &[u8]) -> Result<EventHash, EventError>;
    fn verify_next(&self, event: &EngineEvent, bytes: &[u8]) -> Result<(), EventError>;
    fn advance(&mut self, event_hash: EventHash, sequence: BookSequence) -> Result<(), EventError>;
}
```

Owns chain state and refuses gaps or previous-hash mismatches.

### 35.12 `ReplayEventSource`

```rust
trait ReplayEventSource {
    fn next_verified(&mut self) -> Result<Option<EngineEvent>, EventError>;
}
```

Yields only verified events in deterministic order.

### 35.13 `SnapshotAwareEventSource`

```rust
trait SnapshotAwareEventSource: ReplayEventSource {
    fn start_from_checkpoint(&mut self, checkpoint: ReplayCheckpoint) -> Result<(), EventError>;
}
```

Validates checkpoint hash and sequence before streaming suffix events.

### 35.14 `EventMetricsRecorder`

```rust
trait EventMetricsRecorder {
    fn record_counter(&self, name: &'static str, value: u64, labels: MetricLabels<'_>);
    fn record_histogram(&self, name: &'static str, value: Duration, labels: MetricLabels<'_>);
    fn record_gauge(&self, name: &'static str, value: i64, labels: MetricLabels<'_>);
}
```

Metrics failures must not mutate deterministic state or change event bytes.

## 36. Structs and Enums

### 36.1 `EngineEvent`

Fields: `header: EngineEventHeader`, `payload: EngineEventPayload`. Invariants: header hash fields match canonical payload bytes; event is immutable after construction. Serialization: header first, payload second, schema order only.

### 36.2 `EngineEventHeader`

Fields: `version`, `event_kind`, `book_id`, `book_sequence`, `global_sequence`, `event_id`, `command_id`, `causation_id`, `correlation_id`, `engine_timestamp`, `prev_hash`, `payload_hash`, `event_hash`. Invariants: sequence monotonic per book; `prev_hash` equals current chain state at build time.

### 36.3 `EngineEventPayload`

Enum variants: `OrderAccepted`, `OrderRejected`, `TradeExecuted`, `ClearingResult`, `SettlementJournal`, `CancelAccepted`, `ReplaceAccepted`, `CompensatingAdjustment`, `SnapshotMarker`, `OperationalMarker`. Invariants: discriminants are never reused; unknown variants fail closed unless compatibility adapter supports them.

### 36.4 `EngineEventVersion`

Fields: `major`, `minor`, `patch`, `schema_hash`. Invariants: serializer emits only supported versions; replay declares compatibility before applying.

### 36.5 `ClearingDelta`

Fields: `fill_id`, `book_id`, `product_id`, `buyer`, `seller`, `base_asset`, `quote_asset`, `quantity`, `price`, `notional`, `maker_side`, `taker_side`. Invariants: checked notional; positive quantity; valid maker/taker roles.

### 36.6 `WalletDelta`

Fields: `account_id`, `wallet_id`, `asset_id`, `available_delta`, `held_delta`, `reason`, `source_event_id`, `sequence`. Invariants: fixed-point values; explicit sign; source reference present.

### 36.7 `LedgerDelta`

Fields: `ledger_account_id`, `asset_id`, `debit`, `credit`, `posting_type`, `source_event_id`, `line_id`. Invariants: debit xor credit; totals balance per journal and asset.

### 36.8 `SettlementJournal`

Fields: `journal_id`, `source_event_id`, `book_id`, `book_sequence`, `wallet_deltas`, `ledger_deltas`, `fee_breakdown`, `balance_proof`, `status`. Invariants: immutable, balanced, replayable without current config lookup.

### 36.9 `FeeBreakdown`

Fields: `schedule_version`, `maker_fee`, `taker_fee`, `maker_rebate`, `fee_currency`, `rounding_mode`, `rounding_residue`. Invariants: signed rates use fixed-point; overflow fails.

### 36.10 `ReplayCheckpoint`

Fields: `checkpoint_id`, `book_id`, `last_sequence`, `last_event_hash`, `snapshot_digest`, `schema_version`. Invariants: must reference a verified event hash.

### 36.11 `EventHash`

Fields: fixed-size hash bytes plus algorithm id. Invariants: length matches algorithm; display should be shortened/redacted in logs.

### 36.12 `HashChainState`

Fields: `book_id`, `last_sequence`, `head_hash`, `algorithm`, `domain_separator`. Invariants: advances monotonically only after durable append.

### 36.13 `EventLogEntry`

Fields: `record_length`, `record_checksum`, `event_bytes`, `segment_id`, `offset`, `trailer`. Invariants: segment/offset excluded from event hash.

### 36.14 `ClearingResult`

Fields: `result_id`, `clearing_delta`, `fee_breakdown`, `wallet_deltas`, `ledger_deltas`, `settlement_journal`, `status`. Invariants: all contained deltas refer to the same source decision.

### 36.15 `ClearingError`

Enum variants include arithmetic overflow, invalid fee schedule, invalid role, unbalanced journal, unsupported product, invariant violation, bounded vector exceeded. Serialization: stable code plus safe message.

## 37. Rust-style Pseudocode

```rust
fn build_clearing_delta(fill: &FillFact) -> Result<ClearingDelta, ClearingError> {
    ensure!(fill.quantity > 0, InvalidQuantity);
    let notional = fixed_checked_mul(fill.quantity, fill.price)?;
    Ok(ClearingDelta { fill_id: fill.id, book_id: fill.book_id, product_id: fill.product_id,
        buyer: fill.buyer, seller: fill.seller, base_asset: fill.base_asset,
        quote_asset: fill.quote_asset, quantity: fill.quantity, price: fill.price,
        notional, maker_side: fill.maker_side, taker_side: fill.taker_side })
}

fn calculate_fees(delta: &ClearingDelta, schedule: &FeeSchedule) -> Result<FeeBreakdown, ClearingError> {
    let maker_rate = schedule.rate(delta.product_id, delta.maker_side.account, Role::Maker)?;
    let taker_rate = schedule.rate(delta.product_id, delta.taker_side.account, Role::Taker)?;
    let maker_fee = fixed_round(fixed_checked_mul(delta.notional, maker_rate)?, schedule.rounding)?;
    let taker_fee = fixed_round(fixed_checked_mul(delta.notional, taker_rate)?, schedule.rounding)?;
    Ok(FeeBreakdown::new(schedule.version, maker_fee, taker_fee, schedule.fee_currency))
}

fn build_wallet_delta(delta: &ClearingDelta, fees: &FeeBreakdown) -> Result<Vec<WalletDelta>, ClearingError> {
    let mut out = BoundedVec::new(MAX_WALLET_DELTAS);
    out.push(debit_quote(delta.buyer, delta.notional + fees.buyer_fee()?))?;
    out.push(credit_base(delta.buyer, delta.quantity))?;
    out.push(debit_base(delta.seller, delta.quantity))?;
    out.push(credit_quote(delta.seller, delta.notional - fees.seller_fee()?))?;
    out.push(post_fee_wallet_delta(fees)?)?;
    Ok(out.into_vec())
}

fn build_ledger_delta(delta: &ClearingDelta, fees: &FeeBreakdown) -> Result<Vec<LedgerDelta>, ClearingError> {
    let mut lines = BoundedVec::new(MAX_LEDGER_LINES);
    lines.extend(customer_trade_lines(delta)?)?;
    lines.extend(fee_revenue_lines(fees)?)?;
    ensure_balanced_by_asset(&lines)?;
    Ok(lines.into_vec())
}

fn build_settlement_journal(wallet: Vec<WalletDelta>, ledger: Vec<LedgerDelta>, fees: FeeBreakdown) -> Result<SettlementJournal, ClearingError> {
    ensure_wallet_bounds(&wallet)?;
    ensure_balanced_by_asset(&ledger)?;
    let proof = compute_balance_proof(&ledger)?;
    Ok(SettlementJournal { wallet_deltas: wallet, ledger_deltas: ledger, fee_breakdown: fees, balance_proof: proof, status: JournalStatus::Ready })
}

fn build_engine_event(input: EngineEventBuildInput<'_>) -> Result<EngineEvent, EventError> {
    let payload = EngineEventPayload::ClearingResult(input.clearing_result.clone());
    let payload_bytes = canonical_payload_bytes(&payload)?;
    let payload_hash = hash_payload(&payload_bytes, input.hash_algorithm)?;
    let mut event = EngineEvent::new(input.header_without_hashes, payload, input.prev_hash, payload_hash);
    event.header.event_hash = hash_event_without_current_hash(&event)?;
    Ok(event.freeze())
}

fn serialize_event(event: &EngineEvent, out: &mut BoundedBytes) -> Result<(), EventError> {
    out.clear();
    write_magic(out)?;
    write_version(out, event.header.version)?;
    write_header_fields_in_schema_order(out, &event.header)?;
    write_payload_in_schema_order(out, &event.payload)?;
    Ok(())
}

fn append_event(appender: &mut Appender, event: EngineEvent) -> Result<AppendedEvent, EventError> {
    let mut bytes = BoundedBytes::with_capacity(appender.config.max_record_size);
    appender.serializer.serialize_event(&event, &mut bytes)?;
    appender.hash_chain.verify_next(&event, &bytes)?;
    let record = frame_record(&bytes)?;
    let position = appender.log.append_record(record)?;
    appender.log.apply_durability_policy()?;
    appender.hash_chain.advance(event.header.event_hash, event.header.book_sequence)?;
    Ok(AppendedEvent { event, position })
}

fn verify_hash_chain(source: &mut dyn RawEventSource) -> Result<HashChainState, EventError> {
    let mut state = HashChainState::genesis(source.book_id());
    while let Some(record) = source.next_record()? {
        verify_framing_and_checksum(&record)?;
        let event = deserialize_event(record.bytes())?;
        state.verify_next(&event, record.bytes())?;
        state.advance(event.header.event_hash, event.header.book_sequence)?;
    }
    Ok(state)
}

fn replay_event_stream(source: &mut dyn ReplayEventSource, projections: &mut Projections) -> Result<(), ReplayError> {
    while let Some(event) = source.next_verified()? {
        projections.apply(&event)?;
    }
    Ok(())
}

fn rebuild_state_from_events(source: &mut dyn ReplayEventSource) -> Result<EngineState, ReplayError> {
    let mut state = EngineState::empty();
    while let Some(event) = source.next_verified()? {
        state.apply_event(event)?;
    }
    Ok(state)
}

fn detect_corruption(reader: &mut SegmentReader) -> Result<Option<CorruptionReport>, EventError> {
    let mut verifier = HashChainState::genesis(reader.book_id());
    while let Some(record) = reader.next_raw_record()? {
        if let Err(err) = verify_record_and_chain(&mut verifier, &record) {
            return Ok(Some(CorruptionReport::from_error(err, record.position())));
        }
    }
    Ok(None)
}

fn recover_after_partial_append(log: &mut dyn EventLog, checkpoint: ReplayCheckpoint) -> Result<HashChainState, EventError> {
    let mut verifier = HashChainState::from_checkpoint(checkpoint)?;
    let mut last_good = checkpoint.position();
    for record in log.read_from(last_good)? {
        match verify_record_and_chain(&mut verifier, &record) {
            Ok(()) => last_good = record.end_position(),
            Err(EventError::IncompleteRecord) => break,
            Err(err) => return Err(err),
        }
    }
    log.truncate_to(last_good)?;
    Ok(verifier)
}
```

## 38. Testing Strategy

### 38.1 Unit Tests

| Area | Required cases |
|---|---|
| Fee calculation | maker fee, taker fee, rebate, zero fee, max notional, overflow, unsupported tier, rounding modes. |
| Journal generation | balanced journal, unbalanced rejection, fee revenue line, rebate expense line, wallet-to-ledger correspondence. |
| Serialization | canonical byte order, bounded vector rejection, unknown enum rejection, optional field defaults, golden vectors. |
| Hash generation | genesis hash, next hash, wrong previous hash, payload mutation changes hash, metadata does not change hash. |
| Version compatibility | supported old version decode, unsupported major rejection, added optional field default, reserved discriminant rejection. |

### 38.2 Integration Tests

| Scenario | Expected assertions |
|---|---|
| Accepted order | event appended before success; sequence advances; no clearing delta unless execution occurs. |
| Rejected order | rejection event is auditable; no unauthorized wallet/ledger movement. |
| Partial fill | proportional deltas; held release/remaining hold correct; journal balanced. |
| Full fill | reservation consumed/released; wallet and ledger deltas complete. |
| Cancel | release deltas generated when holds exist; event contains cancellation cause. |
| Replace | old reservation adjustment and new terms represented deterministically. |
| Wallet updates | available/held changes match journal. |
| Ledger updates | debit/credit totals balance by asset. |
| Settlement generation | journal id stable and references source event. |

### 38.3 Replay Tests

| Scenario | Expected assertions |
|---|---|
| Replay from snapshot | checkpoint hash validated; suffix applied; final state matches full replay. |
| Replay from genesis | all events verified and applied in order. |
| Replay after crash | partial tail truncated; last good event preserved. |
| Replay after partial append | incomplete record not yielded; recovery resumes at last good hash. |
| Replay hash verification | corrupted payload/checksum/previous hash stops replay. |

### 38.4 Property Tests

- deterministic serialization for equivalent logical events;
- identical replay state for genesis replay and snapshot replay;
- balance conservation per asset across generated fills;
- immutable event history under attempted mutation;
- stable hash chain under segment rotation and archive/restore;
- bounded vector limits are enforced for generated large inputs.

### 38.5 Performance Tests

- append latency p50/p95/p99/p999 under configured fsync policies;
- serialization latency by payload size;
- replay throughput from hot and archived segments;
- hash generation throughput;
- storage overhead per event and per fill;
- allocation count on clearing and serialization hot paths.

## 39. Replay Validation

Replay validation compares:

- final book state digest;
- wallet projection digest;
- ledger projection digest;
- settlement journal digest;
- last event hash;
- last book sequence.

Divergence is a hard failure. Replay diagnostics should identify the first event where projection digest differs.

## 40. Benchmarks

Benchmark suites must include:

- `clearing_fee_calculation_bench`;
- `clearing_journal_generation_bench`;
- `event_serialization_bench`;
- `event_append_segment_bench`;
- `hash_chain_verify_bench`;
- `replay_from_genesis_bench`;
- `replay_from_snapshot_bench`.

Benchmarks must record input sizes, hardware profile, fsync policy, compiler flags, and schema version.

## 41. Codex Implementation Tasks

1. Create `hermes-clearing` and `hermes-events` crate skeletons only when implementation begins.
2. Add public trait contracts from section 35.
3. Implement fixed-point fee calculation and tests first.
4. Implement wallet/ledger delta builders with balance conservation tests.
5. Implement settlement journal builder and golden examples.
6. Implement `EngineEvent` schema, versioning, and serializer golden vectors.
7. Implement hash-chain state and verifier.
8. Implement segment event log and appender durability policy.
9. Implement replay event sources and snapshot-aware source.
10. Add crash recovery and corruption detection tests.
11. Add metrics/logging/tracing adapters.
12. Add benchmarks and CI gates.
13. Run dependency audit to confirm forbidden hot-path dependencies are absent.

## 42. Review Checklist

- [ ] `hermes-clearing` has no storage, network, Kafka, or clock dependency in deterministic logic.
- [ ] All financial values use fixed-point integer types.
- [ ] Maker/taker role is provided by matching facts and not inferred nondeterministically.
- [ ] Wallet and ledger deltas are explicit and balanced where required.
- [ ] Settlement journals are immutable and reference source events.
- [ ] `EngineEvent` bytes are canonical and golden-vector tested.
- [ ] Event hashes exclude segment metadata and include previous hash linkage.
- [ ] Append happens before externally visible success.
- [ ] Recovery truncates only incomplete active-segment tails.
- [ ] Replay never recalculates historical fees from current schedules.
- [ ] Compatibility code preserves historical hashes.
- [ ] Metrics/logs/traces avoid secret leakage.
- [ ] Benchmarks cover latency, throughput, allocation, and storage overhead.

## 43. Completion Criteria

HIH-005 is implementation-complete when engineers can implement `hermes-clearing`, `hermes-events`, and the ledger/replay boundaries with:

- stable trait contracts;
- explicit module ownership and dependency rules;
- complete data structure invariants;
- deterministic event lifecycle and append pipeline;
- canonical serialization/versioning rules;
- hash-chain verification and recovery procedures;
- replay and snapshot interaction rules;
- concrete pseudocode for core algorithms;
- testing and benchmark matrices;
- review gates that preserve HES principles.
