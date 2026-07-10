# Module Contract: risk-reservation

## Responsibility

`risk-reservation` defines the implementation boundary for HermesNet hot risk admission, reservation lifecycle, credit bucket accounting, risk cache maintenance, funding/liquidation risk deltas, reconciliation, and replay reconstruction.

The module must ensure that an order cannot enter the Book Core hot path unless deterministic local risk has either:

- created or returned an existing replayable reservation; or
- rejected the order with a stable risk reject reason and no active hold.

The module is responsible for preventing:

- double spend;
- double reserve;
- negative available balance;
- unreplayable financial mutation;
- duplicate `client_order_id` creating a second reservation;
- non-idempotent consume/release;
- hot risk state drifting silently from event or cold ledger projections.

## Inputs

Hot-path inputs:

- validated order domain values from gateway/domain layers;
- account ID, symbol ID, asset IDs, side, order type, product type, quantity, price, quote budget, and reduce-only flag;
- immutable configuration snapshot;
- local in-memory credit bucket state;
- local in-memory reservation state;
- local in-memory risk cache state;
- deterministic fee and margin policy snapshots;
- trade consume commands from Book Core/clearing boundary;
- cancel/reject release commands from Book Core/gateway boundary;
- funding and liquidation risk deltas from event-backed upstream modules.

Cold/replay inputs:

- `EngineEvent` views for reservation-created, reservation-consumed, reservation-released, funding-applied, liquidation-applied, and ledger-adjustment events;
- hot risk snapshots if approved by replay policy;
- cold ledger projection views with event watermarks;
- event-derived reservation projections;
- Book Core order state projections for reconciliation.

## Outputs

Hot-path outputs:

- `RiskDecision::Accepted` with `ReservationId` and event-ready reservation delta;
- `RiskDecision::Rejected` with stable `RiskRejectReason` and no active reservation;
- duplicate accepted/rejected decisions that do not mutate balances twice;
- `ReservationDelta` for consume/release/funding/liquidation mutation;
- structured `RiskEngineError` for invariant, arithmetic, capacity, replay, or configuration failure.

Cold/replay outputs:

- reconstructed risk state;
- `ReconciliationResult`;
- `RiskInvariantViolation` reports;
- bounded metrics and structured log fields.

## Public Traits

The `hermes-risk` crate must expose these implementation-grade traits:

- `RiskEngine`: orchestrates order admission, reservation, consume, release, and invariant validation.
- `ReservationStore`: stores reservations and duplicate client-order mappings deterministically.
- `ReservationLifecycle`: applies idempotent consume and release transitions to one reservation.
- `CreditBucketStore`: mutates available/reserved/locked/settled-adjustment amounts.
- `RiskCache`: provides bounded low-latency risk views and drift checks.
- `MarginCalculator`: calculates fixed-point futures margin and reduce-only validation.
- `FundingRiskUpdater`: applies replayable funding risk deltas idempotently.
- `LiquidationRiskUpdater`: applies replayable liquidation risk deltas idempotently.
- `RiskReconciler`: compares reservations, buckets, cache, events, and cold ledger projections.
- `RiskReplay`: reconstructs risk state from event streams.
- `RiskMetricsRecorder`: records bounded metrics without changing risk decisions.

Trait rules:

- all financial values are fixed-point integer types;
- mutating hot traits use `&mut self` and assume single-writer ownership;
- read-only comparison traits use immutable inputs;
- methods return typed results, not stringly typed errors;
- no trait method may require database, Kafka, Redis, HTTP, gRPC, cloud, filesystem, random, or wall-clock access;
- idempotency must be explicit through `client_order_id`, `operation_id`, `event_id`, or event sequence.

## Structs and Enums

The module must define or re-export stable versions of:

- `RiskEngineConfig`;
- `RiskDecision`;
- `RiskRejectReason`;
- `Reservation`;
- `ReservationId`;
- `ReservationState`;
- `ReservationDelta`;
- `ReservationConsume`;
- `ReservationRelease`;
- `CreditBucket`;
- `CreditBucketId`;
- `RiskCacheEntry`;
- `UserRiskState`;
- `AssetRiskState`;
- `MarginRequirement`;
- `FundingRiskDelta`;
- `LiquidationRiskDelta`;
- `ReconciliationResult`;
- `RiskInvariantViolation`;
- `RiskEngineError`.

Struct/enum rules:

- fields that represent money, price, quantity, notional, fee, margin, funding, liquidation, available, reserved, locked, or adjustment values must use fixed-point integer types;
- public error/reject variants must have stable machine-readable codes;
- serialization must be canonical where values enter `EngineEvent`, golden vectors, snapshots, or replay inputs;
- event payloads must include enough data to reconstruct reservation state without hidden process memory;
- user-facing errors must not leak sensitive balances or secrets.

## Crate and Module Layout

Required implementation crate:

```text
crates/hermes-risk/
├── Cargo.toml
├── src/
│   ├── lib.rs
│   ├── engine.rs
│   ├── reservation.rs
│   ├── credit_bucket.rs
│   ├── risk_cache.rs
│   ├── margin.rs
│   ├── funding.rs
│   ├── liquidation.rs
│   ├── reconciliation.rs
│   ├── config.rs
│   ├── error.rs
│   ├── metrics.rs
│   ├── invariants.rs
│   ├── replay.rs
│   └── tests/
│       ├── reservation_tests.rs
│       ├── credit_bucket_tests.rs
│       ├── margin_tests.rs
│       ├── replay_tests.rs
│       ├── property_tests.rs
│       └── performance_tests.rs
```

Module responsibilities:

| Module | Responsibility | Forbidden Behavior |
| --- | --- | --- |
| `lib.rs` | Public API and re-exports. | Side-effectful initialization or global mutable state. |
| `engine.rs` | Risk decision orchestration. | Matching, ledger writes, event-log IO, external calls. |
| `reservation.rs` | Reservation lifecycle and idempotency. | Over-consume, over-release, hidden state. |
| `credit_bucket.rs` | Available/reserved/locked accounting. | Negative balances or unbounded bucket creation. |
| `risk_cache.rs` | Bounded hot risk views and drift detection. | Cache as source of truth over event-backed state. |
| `margin.rs` | Fixed-point margin and reduce-only checks. | Floats or external market-data fetches. |
| `funding.rs` | Funding risk delta application. | Funding rate calculation from floats or clocks. |
| `liquidation.rs` | Liquidation risk delta application. | Silent active-reservation deletion. |
| `reconciliation.rs` | Pure comparison with projections. | Hot-path IO or silent drift suppression. |
| `config.rs` | Validated immutable risk config. | Runtime env/file reads in deterministic logic. |
| `error.rs` | Stable errors and reject reasons. | Panic-based expected control flow. |
| `metrics.rs` | Recorder trait and finite labels. | Blocking exporter dependency. |
| `invariants.rs` | Central invariant checks. | Mutation or IO. |
| `replay.rs` | Event-to-risk-state reconstruction. | Snapshot/event storage IO or nondeterministic ordering. |

## Forbidden Behavior

The module must never:

- use floating-point arithmetic for risk, margin, reservation, fee, funding, liquidation, reconciliation, or replay calculations;
- call a database in the trading hot path;
- call Kafka or any message broker before risk decision;
- call Redis or a remote cache in the hot path;
- call HTTP, gRPC, cloud APIs, or filesystem APIs in deterministic state transitions;
- read wall-clock time to decide risk outcomes;
- generate random IDs for replayable risk state;
- create unbounded queues, maps, vectors, retry buffers, or background mutation loops in the hot path;
- mutate available/reserved/locked balances without replayable event data;
- return external success before required event append protocol is satisfied;
- permit duplicate `client_order_id` to reserve funds twice;
- permit consume or release to apply twice for the same operation;
- treat cold ledger lag as proof of correctness;
- suppress invariant violations as warnings.

## Dependencies Allowed

Allowed production dependencies:

- workspace fixed-point crate for integer financial values;
- workspace ID crate for account, order, client order, reservation, bucket, asset, symbol, event, and operation IDs;
- workspace domain crate for order/product/side/account/symbol views;
- workspace event crate for event payload views and replay contracts;
- small error derive crate if approved by workspace policy;
- optional serialization only for event payloads, snapshots, cold reconciliation, and golden vectors.

Allowed test/benchmark dependencies:

- property testing library;
- benchmark library;
- deterministic fixture helpers;
- workspace golden-vector utilities.

## Dependencies Forbidden

Forbidden in hot-path risk modules:

- SQL clients;
- Kafka or message broker clients;
- Redis or remote cache clients;
- HTTP/gRPC clients;
- cloud SDKs;
- async runtimes;
- floating-point financial libraries;
- wall-clock time crates used in deterministic decisions;
- random number generators used in replayable identity or financial mutation;
- concrete metrics/logging exporters that block or allocate unbounded memory.

## Invariants

| Invariant ID | Requirement |
| --- | --- |
| RISK-INV-001 | `available >= 0`. |
| RISK-INV-002 | `reserved >= 0`. |
| RISK-INV-003 | `locked >= 0`. |
| RISK-INV-004 | `total = available + reserved + locked + settled_adjustments`. |
| RISK-INV-005 | Reservation cannot be consumed more than once for the same operation. |
| RISK-INV-006 | Reservation cannot be released more than once for the same operation. |
| RISK-INV-007 | `consumed + released <= reserved_amount`. |
| RISK-INV-008 | Duplicate `client_order_id` cannot create a second reservation. |
| RISK-INV-009 | Rejected order cannot leave an active reservation. |
| RISK-INV-010 | Cancelled order releases remaining reservation exactly once. |
| RISK-INV-011 | Replay reconstructs the same available/reserved state. |
| RISK-INV-012 | Cold ledger projection eventually reconciles with hot risk state. |
| RISK-INV-013 | Futures margin cannot be negative. |
| RISK-INV-014 | Reduce-only order must not increase exposure. |
| RISK-INV-015 | All financial mutations are auditable and event-backed. |
| RISK-INV-016 | Fee reservation must be conservative and deterministic. |
| RISK-INV-017 | Funding deltas apply once per event ID. |
| RISK-INV-018 | Liquidation deltas apply once per event ID. |
| RISK-INV-019 | Risk cache cannot override authoritative bucket/reservation state. |
| RISK-INV-020 | Reconciliation discrepancies are bounded and stably ordered. |

## Idempotency Rules

Order admission:

- idempotency key is `(account_id, client_order_id, product_type, symbol_id or approved scope)`;
- exact duplicate accepted order returns existing reservation and does not mutate balances;
- conflicting duplicate order returns `DuplicateClientOrderIdConflict` and does not mutate balances;
- duplicate rejected order returns the original reject result when policy/event data supports it;
- idempotency records must be bounded and replayable.

Reservation consume:

- consume command includes `reservation_id`, `operation_id`, and trade identity;
- same operation ID and same payload returns the same result/no-op;
- same operation ID with conflicting payload returns operation conflict;
- consumed amount cannot exceed remaining reservation.

Reservation release:

- release command includes `reservation_id`, `operation_id`, and release reason;
- same operation ID and same payload returns the same result/no-op;
- same operation ID with conflicting payload returns operation conflict;
- release amount cannot exceed remaining reservation;
- cancel/reject/expiry terminal release is applied exactly once.

Funding and liquidation:

- funding applies once per funding event ID;
- liquidation applies once per liquidation event ID;
- duplicate event IDs are idempotent;
- conflicting duplicate event IDs are replay divergence or invariant violation.

## Replay Rules

Replay must:

- start from empty validated config or approved snapshot;
- apply events in durable sequence order;
- verify sequence continuity when sequence metadata is available;
- verify hash chain when hash metadata is available;
- rebuild reservations, credit buckets, idempotency records, applied funding IDs, applied liquidation IDs, and risk cache source state;
- apply reservation-created, reservation-consumed, reservation-released, funding-applied, liquidation-applied, and ledger-adjustment events deterministically;
- validate invariants after each event in strict mode or at deterministic intervals in benchmark mode;
- produce the same available/reserved/locked state as the live engine for the same event prefix.

Replay must not:

- call external services;
- read wall-clock time;
- use random values;
- infer missing event data from current ledger state;
- skip duplicate/idempotency reconstruction.

## Reconciliation Rules

Reservation reconciliation must compare:

- reservation store remaining amounts;
- credit bucket reserved totals;
- risk cache reserved totals;
- event-derived reservation projection;
- active order projection where supplied.

Cold ledger reconciliation must compare:

- compatible sequence watermarks only;
- hot event-derived risk state;
- cold ledger available/reserved/locked projections;
- funding and liquidation adjustments;
- terminal cancel/reject reservation releases.

Risk cache drift detection must:

- compare cache values to authoritative bucket/reservation state;
- report drift with account/asset/bucket/reservation context;
- rebuild or quarantine according to config;
- never allow stale cache data to authorize an order when source state disagrees.

## Test Requirements

Unit tests:

- `RISK-UNIT-001` spot buy reservation;
- `RISK-UNIT-002` spot sell reservation;
- `RISK-UNIT-003` fee reservation;
- `RISK-UNIT-004` market buy quote budget;
- `RISK-UNIT-005` futures margin;
- `RISK-UNIT-006` duplicate reservation prevention;
- `RISK-UNIT-007` release once;
- `RISK-UNIT-008` consume once;
- `RISK-UNIT-009` invariant violation detection.

Integration tests:

- `RISK-INT-001` order accepted and reserved;
- `RISK-INT-002` trade consumes reservation;
- `RISK-INT-003` cancel releases reservation;
- `RISK-INT-004` reject leaves no active reservation;
- `RISK-INT-005` partial fill consume/release;
- `RISK-INT-006` duplicate `client_order_id` retry;
- `RISK-INT-007` funding risk update;
- `RISK-INT-008` liquidation risk update.

Property tests:

- `RISK-PROP-001` no negative available balance;
- `RISK-PROP-002` balance conservation;
- `RISK-PROP-003` no double reserve;
- `RISK-PROP-004` no double release;
- `RISK-PROP-005` no double consume;
- `RISK-PROP-006` replay equivalence.

Replay tests:

- `RISK-REPLAY-001` replay reservation lifecycle;
- `RISK-REPLAY-002` crash after reserve before response;
- `RISK-REPLAY-003` crash after consume before response;
- `RISK-REPLAY-004` duplicate retry after unknown gateway response;
- `RISK-REPLAY-005` cold ledger reconciliation after replay.

Determinism tests:

- same state and same order produce identical `RiskDecision`;
- replay and live execution produce identical buckets and reservations;
- wall-clock changes do not alter output;
- fixed-point rounding vectors remain stable;
- reconciliation discrepancy ordering is stable.

## Benchmark Requirements

Required benchmarks:

- `RISK-BENCH-001` `reserve_for_order` latency;
- `RISK-BENCH-002` reservation consume latency;
- `RISK-BENCH-003` release latency;
- `RISK-BENCH-004` risk cache lookup latency;
- `RISK-BENCH-005` reconciliation batch throughput;
- `RISK-BENCH-006` replay throughput;
- `RISK-BENCH-007` hot-path allocation count;
- `RISK-BENCH-008` bounded-capacity pressure behavior.

Benchmark rules:

- run with release settings;
- use deterministic fixtures;
- track p50, p95, p99, and p999 where useful;
- fail CI on forbidden allocation or dependency regressions if tooling supports it;
- publish baseline thresholds with hardware/compiler metadata.

## Related HES References

- `HES/HES-003 Volume III.md` for Book Core boundary and event ordering.
- `HES/HES-004 Volume IV.md` for risk, reservation, and credit bucket obligations.
- `HES/HES-005 Volume V.md` for EngineEvent and settlement event rules.
- `HES/HES-006 Volume VI.md` for wallet/ledger projection reconciliation.
- `HES/HES-007 Volume VII.md` for futures, margin, funding, liquidation, and reduce-only behavior.
- `HES/HES-009 Volume IX.md` for deterministic replay and testing.
- `HES/ADR/ADR-0005-fixed-point-arithmetic.md`.
- `HES/ADR/ADR-0006-event-sourcing.md`.
- `HES/ADR/ADR-0007-no-database-in-hot-path.md`.
- `HES/ADR/ADR-0008-no-kafka-before-decision.md`.
- `HES/ADR/ADR-0011-credit-bucket-risk-model.md`.
- `HES/ADR/ADR-0013-idempotent-client-order-id.md`.
- `HES/ADR/ADR-0014-deterministic-replay.md`.
- `HES/ADR/ADR-0015-hot-cold-path-separation.md`.
- `HES/ADR/ADR-0016-hash-chained-engine-events.md`.

## Completion Checklist

- [ ] `hermes-risk` crate skeleton matches required file layout.
- [ ] All required traits are defined with deterministic, idempotent contracts.
- [ ] All required structs/enums exist with fixed-point financial fields.
- [ ] Spot buy/sell reservation paths are implemented and tested.
- [ ] Market buy/sell reservation paths are budget-bounded and tested.
- [ ] Fee reservation is conservative and fixed-point only.
- [ ] Futures margin reservation is fixed-point only and tested.
- [ ] Reduce-only orders cannot increase exposure.
- [ ] Funding risk updates are event-backed and idempotent.
- [ ] Liquidation risk updates are event-backed and idempotent.
- [ ] Duplicate `client_order_id` cannot double reserve.
- [ ] Consume and release are idempotent.
- [ ] Explicit invariant table is implemented in tests.
- [ ] Replay reconstructs identical reservation and bucket state.
- [ ] Cold ledger reconciliation handles clean, lagged, and drift outcomes.
- [ ] Risk cache drift detection reports bounded structured discrepancies.
- [ ] Metrics labels are bounded and do not affect decisions.
- [ ] Logs are structured and avoid sensitive data leakage.
- [ ] Dependency audit proves no forbidden hot-path dependencies.
- [ ] Benchmarks cover reserve, consume, release, cache lookup, reconciliation, and replay.
- [ ] Review confirms no HES behavior was expanded or contradicted.
