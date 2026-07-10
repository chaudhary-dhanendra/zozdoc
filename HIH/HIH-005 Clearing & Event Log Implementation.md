# Clearing & Event Log Implementation

## 1. Purpose

Define implementation-grade Rust guidance for clearing & event log implementation while preserving HES as the source of truth. This file specifies crate responsibilities, module boundaries, contracts, deterministic control flow, pseudocode, tests, benchmarks, and review gates.

## 2. Related HES volumes

- HES volumes for matching, risk, clearing, settlement, events, replay, operations, observability, and security.
- HES invariants: fixed-point integers only; no floats in matching/risk/settlement; bounded queues; immutable hash-chained EngineEvents; event append before externally visible success; deterministic replay; no DB/Kafka in the hot path; all financial mutations auditable.

## 3. Implementation scope

Covers clearing delta creation, spot/futures clearing, maker/taker fees, rebates, wallet and ledger deltas, journals, EngineEvent schema/builder, binary serialization, hash chain, fsync, versioning, compatibility, and corruption detection.

## 4. Non-goals

- No production Rust source files are created by this handbook.
- No expansion of HES behavior or changes to HES obligations.
- No DOCX/PDF generation.
- No unbounded caches, hidden threads, wall-clock ordering, random IDs, floats, SQL, or Kafka in hot-path modules.

## 5. Target crates

- `hermes-clearing`
- `hermes-events`
- `hermes-ledger`
- `hermes-fixed`
- `hermes-replay`

## 6. Module layout

Use `lib.rs` for public contracts and re-exports. Use crate-local `config.rs`, `error.rs`, `types.rs`, `metrics.rs`, `tests/`, and domain modules. Hot modules receive dependencies as traits and own their mutable state. Cold modules perform IO, config loading, process supervision, and external protocol adaptation.

## 7. File layout

```text
crates/
  hermes-clearing/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-events/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-ledger/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-fixed/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-replay/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
```

## 8. Key traits

- `ClearingEngine`; `FeeCalculator`; `EventBuilder`; `EventSerializer`; `EventLog`; `HashChainVerifier`; `JournalBuilder`.

## 9. Key structs/enums

`ClearingDelta`, `WalletDelta`, `LedgerDelta`, `SettlementJournal`, `FeeSchedule`, `FeeAmount`, `EngineEventHeader`, `EngineEventBody`, `SerializedEvent`, `HashChainLink`, `LogSegment`, `EventVersion`.

## 10. Error types

Use domain-specific error enums with stable reject/recovery codes, e.g. `InvalidCommand`, `ArithmeticOverflow`, `InvariantViolation`, `Duplicate`, `InsufficientAvailable`, `QueueFull`, `EventAppendFailed`, `HashChainMismatch`, `SnapshotInvalid`, `ReplayDiverged`, `ExternalDependencyUnavailable`, and `ConfigurationInvalid`. Errors must include machine-actionable codes and avoid secret/account-sensitive debug leakage.

## 11. Control flow

Matching fills become clearing deltas: spot transfers base/quote, futures update positions/collateral/realized PnL, fees/rebates post to revenue or maker rebate accounts. The EventBuilder assembles command, matches, risk holds, clearing deltas, ledger deltas, and metadata into an immutable versioned EngineEvent. Serialization is canonical binary. EventLog appends with previous hash, length, CRC/checksum, and fsync policy before success. Replay validates version compatibility, length, checksum, hash chain, and monotonic sequence; corruption stops at first invalid record and requires recovery procedure.

## 12. Rust-style pseudocode

```rust
fn build_clearing_delta(fill){ match fill.product { Spot=>spot_wallet_ledger_deltas(fill), Futures=>futures_position_collateral_deltas(fill) } }
fn calculate_fee(fill, role){ rate=fee_schedule.lookup(fill.account, role); fixed_mul(fill.notional, rate) }
fn build_engine_event(parts){ EventBuilder::new(version).with_header(seq,prev_hash).with_body(parts).build_immutable() }
fn serialize_event(ev){ canonical_binary_encode(ev); append_len_crc_hash(encoded) }
fn append_event(log, ev){ rec=serialize_event(ev); log.write(rec); log.apply_fsync_policy(); log.advance_head(ev.hash); }
fn verify_hash_chain(events){ prev=genesis; for e in events { assert_eq!(e.prev_hash,prev); assert_eq!(hash(e),e.hash); prev=e.hash; } }
fn rebuild_from_events(events){ verify_hash_chain(events); for e in events { projections.apply(e); } }
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
