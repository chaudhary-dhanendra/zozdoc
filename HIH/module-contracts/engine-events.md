# Module Contract: engine-events

## Responsibility

`hermes-events` owns immutable `EngineEvent` construction, canonical binary serialization, append-only event logging, hash-chain state, version compatibility, verified replay event sources, corruption detection, and crash recovery. It is the durability boundary for externally visible engine success.

## Interfaces

### Public traits

```rust
trait EngineEventBuilder {
    fn build_engine_event(&self, input: EngineEventBuildInput<'_>) -> Result<EngineEvent, EventError>;
}

trait EngineEventSerializer {
    fn serialize_event(&self, event: &EngineEvent, out: &mut BoundedBytes) -> Result<(), EventError>;
    fn deserialize_event(&self, bytes: &[u8]) -> Result<EngineEvent, EventError>;
}

trait EventLog {
    fn append_record(&mut self, record: EventLogRecord<'_>) -> Result<EventLogPosition, EventError>;
    fn read_from(&self, position: EventLogPosition) -> Result<EventLogIterator<'_>, EventError>;
    fn truncate_to(&mut self, position: EventLogPosition) -> Result<(), EventError>;
}

trait EventAppender {
    fn append_event(&mut self, event: EngineEvent) -> Result<AppendedEvent, EventError>;
}

trait HashChain {
    fn current_state(&self) -> HashChainState;
    fn compute_event_hash(&self, event: &EngineEvent, bytes: &[u8]) -> Result<EventHash, EventError>;
    fn verify_next(&self, event: &EngineEvent, bytes: &[u8]) -> Result<(), EventError>;
    fn advance(&mut self, event_hash: EventHash, sequence: BookSequence) -> Result<(), EventError>;
}

trait ReplayEventSource {
    fn next_verified(&mut self) -> Result<Option<EngineEvent>, EventError>;
}

trait SnapshotAwareEventSource: ReplayEventSource {
    fn start_from_checkpoint(&mut self, checkpoint: ReplayCheckpoint) -> Result<(), EventError>;
}

trait EventMetricsRecorder {
    fn record_counter(&self, name: &'static str, value: u64, labels: MetricLabels<'_>);
    fn record_histogram(&self, name: &'static str, value: Duration, labels: MetricLabels<'_>);
    fn record_gauge(&self, name: &'static str, value: i64, labels: MetricLabels<'_>);
}
```

### Core structs

- `EngineEvent`: immutable header plus payload.
- `EngineEventHeader`: version, kind, book id, sequence, event id, command/causation/correlation ids, deterministic timestamp, previous hash, payload hash, event hash.
- `EngineEventPayload`: accepted/rejected order, trade, clearing result, settlement journal, cancel, replace, compensating adjustment, snapshot marker, operational marker.
- `EngineEventVersion`: major, minor, patch, schema hash.
- `EventHash`: fixed-size hash bytes plus algorithm id.
- `HashChainState`: book id, last sequence, head hash, algorithm, domain separator.
- `EventLogEntry`: record length, checksum, canonical event bytes, segment id, byte offset, trailer.
- `ReplayCheckpoint`: checkpoint id, book id, last sequence, last event hash, snapshot digest, schema version.
- `EventError`: stable error code for serialization, append, unsupported version, hash mismatch, corruption, and recovery cases.

## Invariants

- `EngineEvent` is immutable after hash calculation.
- Canonical bytes are the hash source of truth.
- Segment path, offset, archive location, and metrics metadata are excluded from event hash.
- Book sequence is monotonic and gap-free within a stream.
- `prev_hash` equals the current chain head at append time.
- Event append completes before externally visible success.
- Partial records are not yielded as committed records.
- Historical bytes are never rewritten; fixes use compensating events.

## Allowed dependencies

- Hermes fixed-point and identifier crates.
- Approved byte buffer and checksum/hash crates.
- Local schema, versioning, config, error, and metrics modules.
- Filesystem access in event log modules only.

## Forbidden dependencies

- Database clients in append or replay hot path.
- Kafka/message broker publication before append decision.
- Wall-clock reads for event ordering.
- Random hash salts or nondeterministic serialization.
- Unbounded record sizes or unbounded decode allocation.
- In-place mutation of historical event bytes.

## Replay guarantees

`ReplayEventSource` yields only records that pass framing, checksum, decode, version, payload hash, event hash, previous-hash, and sequence verification. `SnapshotAwareEventSource` validates the checkpoint hash and starts at the first suffix event after the checkpoint sequence. Replay consumers must be able to rebuild identical state from genesis or snapshot plus suffix.

## Audit guarantees

Every event records the source command, causation, book-local sequence, payload, previous hash, current hash, and schema version. Auditors can verify original bytes, hash chain, fee amounts, wallet deltas, ledger deltas, and settlement journal without trusting mutable application logs.

## Benchmark expectations

- Serialization latency and allocation count for representative payloads.
- Append latency under each supported fsync policy.
- Hash generation and verification throughput.
- Replay throughput from genesis and snapshot.
- Recovery scan time for active segment with partial tail.
- Storage overhead per event and per segment.

## HES references

Relevant HES areas: events, replay, matching/book ordering, clearing/settlement audit, operations, observability, and security. HES principles enforced here include immutable hash-chained EngineEvents, append-only log, deterministic replay, no database dependency in the hot path, no Kafka before decision durability, and append-before-success.

## Implementation checklist

- [ ] Define `EngineEvent` schema and reserve enum discriminants.
- [ ] Implement canonical binary serializer and golden vectors.
- [ ] Implement event version compatibility matrix.
- [ ] Implement hash-chain compute, verify, and advance.
- [ ] Implement framed segment event log with checksum and truncation recovery.
- [ ] Implement appender pipeline with durability fence before success.
- [ ] Implement verified replay and snapshot-aware event sources.
- [ ] Add corruption detection reports.
- [ ] Add metrics/logging/tracing with safe labels.
- [ ] Add benchmarks and dependency audit for forbidden hot-path dependencies.
