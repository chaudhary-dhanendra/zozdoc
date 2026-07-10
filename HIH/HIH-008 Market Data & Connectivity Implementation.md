# Market Data & Connectivity Implementation

## 1. Purpose

Define implementation-grade Rust guidance for market data & connectivity implementation while preserving HES as the source of truth. This file specifies crate responsibilities, module boundaries, contracts, deterministic control flow, pseudocode, tests, benchmarks, and review gates.

## 2. Related HES volumes

- HES volumes for matching, risk, clearing, settlement, events, replay, operations, observability, and security.
- HES invariants: fixed-point integers only; no floats in matching/risk/settlement; bounded queues; immutable hash-chained EngineEvents; event append before externally visible success; deterministic replay; no DB/Kafka in the hot path; all financial mutations auditable.

## 3. Implementation scope

Covers projectors, public/private WebSockets, REST market endpoints, FIX, OUCH, SBE, snapshots, deltas, sequence numbers, checksums, gaps, reconnect, backpressure, drop copy, and execution reports.

## 4. Non-goals

- No production Rust source files are created by this handbook.
- No expansion of HES behavior or changes to HES obligations.
- No DOCX/PDF generation.
- No unbounded caches, hidden threads, wall-clock ordering, random IDs, floats, SQL, or Kafka in hot-path modules.

## 5. Target crates

- `hermes-market-data`
- `hermes-connectivity`
- `hermes-gateway`
- `hermes-events`
- `hermes-sbe`
- `hermes-fix`

## 6. Module layout

Use `lib.rs` for public contracts and re-exports. Use crate-local `config.rs`, `error.rs`, `types.rs`, `metrics.rs`, `tests/`, and domain modules. Hot modules receive dependencies as traits and own their mutable state. Cold modules perform IO, config loading, process supervision, and external protocol adaptation.

## 7. File layout

```text
crates/
  hermes-market-data/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-connectivity/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-gateway/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-events/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-sbe/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-fix/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
```

## 8. Key traits

- `MarketDataProjector`; `SnapshotBuilder`; `DeltaPublisher`; `WebSocketStream`; `FixSession`; `SbeEncoder`; `GapRecoveryService`; `DropCopyPublisher`.

## 9. Key structs/enums

`OrderBookSnapshot`, `BookDelta`, `TradeDelta`, `StreamSequence`, `Checksum`, `Subscription`, `ReconnectToken`, `FixExecutionReport`, `SbeFrame`, `DropCopyEvent`, `SlowConsumerPolicy`.

## 10. Error types

Use domain-specific error enums with stable reject/recovery codes, e.g. `InvalidCommand`, `ArithmeticOverflow`, `InvariantViolation`, `Duplicate`, `InsufficientAvailable`, `QueueFull`, `EventAppendFailed`, `HashChainMismatch`, `SnapshotInvalid`, `ReplayDiverged`, `ExternalDependencyUnavailable`, and `ConfigurationInvalid`. Errors must include machine-actionable codes and avoid secret/account-sensitive debug leakage.

## 11. Control flow

Projectors consume durable EngineEvents, build public book/trade streams and private execution reports, and assign stream sequences separate from book-local sequence while preserving mapping. Snapshots contain sequence and checksum; deltas include prev/current sequence. Clients detect gaps and resubscribe/recover from snapshots. REST market endpoints read projections, not Book Core. FIX sessions and OUCH boundaries normalize/emit execution reports. SBE encoders use versioned schemas. Slow consumers use bounded outboxes, drop/replay policies, and disconnect thresholds.

## 12. Rust-style pseudocode

```rust
fn publish_trade_delta(ev){ delta=TradeDelta::from_event(ev); delta.seq=next_stream_seq(); publisher.broadcast(delta); }
fn build_order_book_snapshot(book){ levels=top_n(book); checksum=crc(levels); return Snapshot(seq,last_book_seq,levels,checksum); }
fn apply_sequence_gap_detection(client,msg){ if msg.prev_seq != client.last_seq { recovery.request_snapshot(client); } }
fn handle_client_resubscribe(c, token){ snap=snapshot_builder.build(token.symbol); send(c,snap); replay_deltas_after(c,snap.seq); }
fn encode_sbe_message(msg){ schema=select_schema_version(msg); return sbe.encode(schema,msg); }
fn publish_execution_report(ev){ report=FixExecutionReport::from_event(ev); drop_copy.publish(report); private_ws.publish(report); }
```

## 13. Configuration

Fixed-point scales, queue bounds, snapshot intervals, fsync policy, feature flags, and environment-specific limits must be explicit and validated at startup.

## 14. Observability hooks

Expose counters, gauges, histograms, structured logs, and spans for command count, reject reason, durable sequence, replay position, queue depth, allocation count where relevant, snapshot age, hash-chain head, and p50/p95/p99/p999 latency. Logs must be event-correlatable and must not contain secrets.

## 15. Failure handling

Fail closed on invariant violations. Reject before mutation when validation fails. If an event append fails after local computation, do not publish success; stop or quarantine the writer and require replay recovery. Recovery always starts from the last verified snapshot plus verified hash-chain suffix.

## 16. Testing strategy

Unit tests cover arithmetic, validation, state transitions, error mapping, and invariant preservation. Property tests generate command sequences and assert no negative balances, deterministic ordering, sequence monotonicity, and replay equivalence. Integration tests exercise gateway-to-event or event-to-projection paths. Golden vectors lock serialization and accounting outputs.

## 17. Benchmark strategy

Measure steady-state throughput, p99/p999 latency, allocation counts, memory footprint, queue pressure, snapshot/replay speed, and overload behavior. Hot-path benchmarks must run with release settings and detect accidental allocation regressions.

## 18. Codex implementation tasks

1. Create crate skeletons and public trait contracts.
2. Add fixed-point/ID/domain dependencies only in allowed directions.
3. Implement pure state transition modules before IO adapters.
4. Add serialization/golden-vector tests.
5. Add replay/property tests.
6. Add benchmarks and CI gates.
7. Perform dependency and unsafe-code audit.

## 19. Review checklist

- [ ] HES invariants are preserved.
- [ ] Fixed-point types are used for every financial value.
- [ ] Queues/caches/pools are bounded.
- [ ] Events are append-only, immutable, versioned, and replayable.
- [ ] No forbidden hot-path dependency is present.
- [ ] Failure paths are deterministic and auditable.
- [ ] Tests and benchmarks cover documented invariants.

## 20. Implementation completion criteria

Complete when the documented crates have stable contracts, deterministic state transitions, replay/golden-vector tests, benchmark baselines, observability hooks, failure handling, and review evidence sufficient for Codex or engineers to start production Rust implementation.
