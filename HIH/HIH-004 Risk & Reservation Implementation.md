# Risk & Reservation Implementation

## 1. Purpose

Define implementation-grade Rust guidance for risk & reservation implementation while preserving HES as the source of truth. This file specifies crate responsibilities, module boundaries, contracts, deterministic control flow, pseudocode, tests, benchmarks, and review gates.

## 2. Related HES volumes

- HES volumes for matching, risk, clearing, settlement, events, replay, operations, observability, and security.
- HES invariants: fixed-point integers only; no floats in matching/risk/settlement; bounded queues; immutable hash-chained EngineEvents; event append before externally visible success; deterministic replay; no DB/Kafka in the hot path; all financial mutations auditable.

## 3. Implementation scope

Covers reservation lifecycle, quote/base/fee holds, futures margin, credit buckets, retail sharding, funding/liquidation updates, replay reconstruction, and reconciliation.

## 4. Non-goals

- No production Rust source files are created by this handbook.
- No expansion of HES behavior or changes to HES obligations.
- No DOCX/PDF generation.
- No unbounded caches, hidden threads, wall-clock ordering, random IDs, floats, SQL, or Kafka in hot-path modules.

## 5. Target crates

- `hermes-risk`
- `hermes-domain`
- `hermes-fixed`
- `hermes-events`
- `hermes-ledger`

## 6. Module layout

Use `lib.rs` for public contracts and re-exports. Use crate-local `config.rs`, `error.rs`, `types.rs`, `metrics.rs`, `tests/`, and domain modules. Hot modules receive dependencies as traits and own their mutable state. Cold modules perform IO, config loading, process supervision, and external protocol adaptation.

## 7. File layout

```text
crates/
  hermes-risk/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-domain/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-fixed/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-events/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-ledger/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
```

## 8. Key traits

- `RiskEngine`; `ReservationStore`; `CreditBucketStore`; `MarginCalculator`; `RiskCache`; `ReservationReconciler`.

## 9. Key structs/enums

`Reservation`, `Hold`, `CreditBucket`, `RiskAccount`, `AvailableBalance`, `MarginRequirement`, `RiskDecision`, `ClientOrderKey`, `RiskInvariantReport`.

## 10. Error types

Use domain-specific error enums with stable reject/recovery codes, e.g. `InvalidCommand`, `ArithmeticOverflow`, `InvariantViolation`, `Duplicate`, `InsufficientAvailable`, `QueueFull`, `EventAppendFailed`, `HashChainMismatch`, `SnapshotInvalid`, `ReplayDiverged`, `ExternalDependencyUnavailable`, and `ConfigurationInvalid`. Errors must include machine-actionable codes and avoid secret/account-sensitive debug leakage.

## 11. Control flow

Reserve before matching: spot buys reserve quote plus fee, spot sells reserve base, futures reserve initial/maintenance margin and fees. Consume holds on fills, release on rejects/cancels/expiry. Duplicate `client_order_id` returns the existing reservation/result. Risk cache mirrors ledger-derived balances plus open holds; credit buckets support market-maker and retail-sharded limits. Reconciliation compares event-derived holds to ledger projections and enforces no-negative-available. Funding and liquidation events update margin availability via replayable deltas.

## 12. Rust-style pseudocode

```rust
fn reserve_for_order(o){ if duplicate(o.client_order_id){return existing()} match o.product { Spot=>if o.side==Buy{reserve_spot_buy(o)}else{reserve_spot_sell(o)}, Futures=>reserve_futures_margin(o)} }
fn reserve_spot_buy(o){ quote=o.price*o.qty; fee=max_fee(o); store.hold(o.account, quote_asset, quote+fee) }
fn reserve_spot_sell(o){ store.hold(o.account, base_asset, o.qty) }
fn reserve_futures_margin(o){ req=margin.initial(o.notional, o.leverage); store.hold(o.account, collateral, req+fee(o)) }
fn consume_reservation(fill){ store.consume(fill.hold_id, fill.settled_amount); }
fn release_reservation(reason, hold){ store.release_remaining(hold, reason); }
fn reconcile_reservations(){ for acct in accounts { assert_eq!(event_holds(acct), ledger_holds(acct)); validate_risk_invariants(acct); } }
fn validate_risk_invariants(acct){ assert!(available(acct) >= 0); assert!(holds(acct) >= consumed(acct)); }
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
