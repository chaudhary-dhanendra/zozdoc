# Risk & Reservation Implementation

## 1. Purpose

HIH-004 defines the implementation handbook for the `hermes-risk` crate and its reservation, credit bucket, risk cache, margin, funding, liquidation, reconciliation, and replay modules.

The chapter is intentionally concrete. A Rust engineer or Codex agent should be able to create the crate skeleton, public contracts, tests, benchmarks, and first implementation pass from this document without inventing risk semantics.

The risk and reservation subsystem has one primary job: decide whether an order may cross the Book Core boundary and, if accepted, atomically reserve enough user balance or margin so the matching engine cannot create double spend, negative available balance, or unreplayable financial state.

The subsystem must preserve these rules:

- fixed-point integers only;
- deterministic risk decisions;
- no floating point in risk, margin, fee, or reservation logic;
- no database, Kafka, cloud service, HTTP call, RPC call, or filesystem dependency in the trading hot path;
- the risk decision is local to the Book Core hot path boundary;
- no double spend;
- no double reserve;
- no negative available balance;
- reservation consume and release are idempotent;
- duplicate `client_order_id` cannot double reserve;
- all risk mutations produce replayable `EngineEvent` data;
- cold ledger projection reconciles with hot risk state;
- all financial mutations are auditable;
- replay reconstructs identical reservation state.

This document does not create production source files. It describes the source files to create in the implementation phase.

## 2. Related HES Volumes

Risk and reservation implementation must remain subordinate to HES. If this handbook appears to conflict with HES, HES wins.

Related material:

- `HES/HES-001 Volume I.md` for core system principles, deterministic state, and product boundaries.
- `HES/HES-002 Volume II.md` for gateway and order ingress semantics.
- `HES/HES-003 Volume III.md` for Book Core matching boundary, event ordering, and local risk handoff.
- `HES/HES-004 Volume IV.md` for risk, reservation, credit buckets, and ledger-facing financial invariants.
- `HES/HES-005 Volume V.md` for clearing, events, hash chain, and settlement event semantics.
- `HES/HES-006 Volume VI.md` for wallet, ledger, and cold projection reconciliation.
- `HES/HES-007 Volume VII.md` for futures, funding, liquidation, margin, and reduce-only semantics.
- `HES/HES-009 Volume IX.md` for deterministic replay, testing, golden vectors, and verification.
- `HES/ADR/ADR-0005-fixed-point-arithmetic.md` for fixed-point arithmetic requirements.
- `HES/ADR/ADR-0006-event-sourcing.md` for event-sourced replay requirements.
- `HES/ADR/ADR-0007-no-database-in-hot-path.md` for hot-path dependency limits.
- `HES/ADR/ADR-0008-no-kafka-before-decision.md` for pre-decision messaging limits.
- `HES/ADR/ADR-0011-credit-bucket-risk-model.md` for bucket-level hot risk state.
- `HES/ADR/ADR-0013-idempotent-client-order-id.md` for duplicate order handling.
- `HES/ADR/ADR-0014-deterministic-replay.md` for replay equivalence.
- `HES/ADR/ADR-0015-hot-cold-path-separation.md` for local hot risk and cold reconciliation separation.
- `HES/ADR/ADR-0016-hash-chained-engine-events.md` for immutable event append.

## 3. Implementation Scope

HIH-004 covers the `hermes-risk` crate only.

In scope:

- hot-path order risk decision;
- reservation lifecycle for accepted orders;
- idempotent duplicate `client_order_id` handling;
- spot buy quote reservation;
- spot sell base reservation;
- market buy quote budget reservation;
- market sell base reservation;
- fee reservation and fee cap handling;
- futures initial margin reservation;
- maintenance margin checks where required for order acceptance;
- reduce-only risk checks;
- funding risk deltas applied to hot risk state;
- liquidation risk deltas applied to hot risk state;
- credit bucket state transitions;
- risk cache reads and writes;
- reservation consume on trade execution;
- reservation release on cancel, reject, expiry, self-trade-prevention reject, and replace rejection;
- reconciliation between reservation store, risk cache, event stream, and cold ledger projections;
- replay reconstruction from `EngineEvent` data;
- invariant validation;
- metrics and logs that do not affect deterministic decisions;
- unit, integration, property, replay, determinism, and benchmark requirements.

Out of scope but referenced:

- production Book Core matching code;
- production ledger persistence code;
- production event log storage code;
- production gateway retry logic;
- production Rust file generation in this task.

## 4. Non-Goals

This chapter must not be used to expand HES or to bypass HES.

Non-goals:

- no production Rust source files are created by this document update;
- no DOCX or PDF generation;
- no new product semantics beyond HES;
- no dependency on SQL, Kafka, Redis, object storage, cloud APIs, gRPC, REST, or filesystem operations in the hot decision path;
- no floating-point arithmetic in any risk, fee, margin, funding, liquidation, reservation, reconciliation, or replay calculation;
- no wall-clock reads in deterministic state transitions;
- no random IDs in deterministic state transitions;
- no best-effort accounting mutation that lacks replayable event data;
- no reservation mutation that cannot be replayed exactly;
- no cache update without an invariant check or event-backed recovery path;
- no hidden background thread that mutates hot risk state outside the single-writer boundary;
- no unbounded map, queue, vector, or retry buffer in the hot path;
- no production ledger settlement inside `hermes-risk`;
- no balance mutation that skips cold ledger reconciliation.

## 5. Risk & Reservation Design Summary

The hot risk path sits between validated order ingress and Book Core matching.

For each order, `RiskEngine::reserve_for_order` performs deterministic checks and either:

1. rejects the order with a stable `RiskRejectReason`, producing no active reservation; or
2. accepts the order and creates or returns an existing active reservation tied to the order identity.

Accepted orders enter Book Core only after risk has reserved the required amount. The required amount depends on product and side:

- spot buy limit: reserve quote notional plus maximum fee;
- spot sell limit: reserve base quantity plus any fee asset amount if fee is not paid from proceeds;
- market buy: reserve an explicit quote budget plus maximum fee;
- market sell: reserve base quantity plus any fee asset amount if required;
- futures open/increase: reserve initial margin plus fee and verify maintenance constraints;
- futures reduce-only: verify the order cannot increase exposure and normally avoid new initial margin reservation except bounded fee handling.

Risk state is local, deterministic, and event-backed. A reservation is not considered externally durable until the matching/event boundary emits the corresponding `EngineEvent` data according to the event append-before-success rule. Replay must reconstruct identical reservation and credit bucket state by applying the same sequence of events.

Cold ledger state is authoritative for long-term accounting, statements, and audit. Hot risk state is authoritative only for immediate trading admission. Reconciliation verifies that the two eventually converge and that no hot reservation can silently diverge from ledger projections.

## 6. Crate Boundary

The concrete crate is `crates/hermes-risk`.

`hermes-risk` owns:

- risk decision orchestration;
- reservation state machine;
- credit bucket state;
- risk cache state;
- margin requirement calculation contracts;
- funding and liquidation risk delta application;
- reconciliation contracts and pure comparison logic;
- replay reconstruction contracts and pure event application logic;
- risk-specific errors, invariants, metrics labels, and configuration.

`hermes-risk` does not own:

- Book Core matching algorithms;
- durable event log implementation;
- wallet ledger persistence;
- gateway authentication;
- market data publication;
- network protocols;
- production database access;
- Kafka or queue adapters;
- filesystem snapshots.

Allowed crate dependencies:

- `hermes-fixed` for fixed-point integer money, quantity, price, ratio, and basis-point types;
- `hermes-ids` for deterministic account, order, client order, symbol, asset, reservation, event, and bucket identifiers;
- `hermes-domain` for order, side, product, account, asset, symbol, and execution domain types;
- `hermes-events` for event payload contracts and replay event views;
- `serde` only behind explicit feature gates for cold serialization and golden-vector tests;
- `thiserror` or equivalent error derivation if approved by workspace policy;
- `proptest` and benchmark crates in test/bench contexts only.

Forbidden dependencies:

- `f32`, `f64`, decimal libraries that internally round nondeterministically, or any float conversion;
- SQL clients;
- Kafka clients;
- Redis clients;
- HTTP/gRPC clients;
- cloud SDKs;
- wall-clock time sources inside deterministic functions;
- random number generators inside deterministic functions;
- async runtimes in hot-path state transition modules;
- global mutable singletons;
- unbounded concurrent queues.

## 7. Module Layout

Required crate layout:

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

Module rules:

- `engine.rs` orchestrates only. It should delegate arithmetic to reservation, margin, and funding modules.
- `reservation.rs` owns the reservation state machine and idempotency rules.
- `credit_bucket.rs` owns per-bucket availability and reserved/locked accounting.
- `risk_cache.rs` owns query-optimized user/asset state and drift checks.
- `margin.rs` owns deterministic margin calculations and reduce-only checks.
- `funding.rs` owns funding risk delta validation and application.
- `liquidation.rs` owns liquidation risk delta validation and application.
- `reconciliation.rs` owns pure reconciliation comparisons, not IO.
- `config.rs` owns validated configuration types.
- `error.rs` owns stable error and reject reason types.
- `metrics.rs` owns metric labels and recorder trait, not global telemetry clients.
- `invariants.rs` owns invariant validation.
- `replay.rs` owns event-to-state reconstruction logic.

## 8. File Layout

### `Cargo.toml`

Responsibility:

- declare crate metadata, features, dependencies, dev-dependencies, and benchmark/test feature flags.

Public items:

- no Rust public items.

Forbidden behavior:

- no hot-path dependency on DB, Kafka, Redis, cloud SDK, HTTP, gRPC, async runtime, or filesystem-only utility crate;
- no default feature that enables serialization if workspace policy requires `no_std`-like hot profiles;
- no dependency that pulls floating-point financial arithmetic into risk modules.

Allowed dependencies:

- workspace-local domain, fixed, ids, and event crates;
- error derive crate;
- optional `serde` under `serde` feature;
- testing and benchmark dependencies under dev-dependencies.

Forbidden dependencies:

- runtime IO clients;
- nondeterministic RNG for production logic;
- crates that require wall-clock time in core logic.

Tests required:

- dependency audit test or CI check proving forbidden crates are absent.

### `src/lib.rs`

Responsibility:

- define crate-level documentation;
- re-export public traits, structs, enums, and module namespaces;
- keep public API stable and minimal.

Public items:

- `RiskEngine`;
- `ReservationStore`;
- `ReservationLifecycle`;
- `CreditBucketStore`;
- `RiskCache`;
- `MarginCalculator`;
- `FundingRiskUpdater`;
- `LiquidationRiskUpdater`;
- `RiskReconciler`;
- `RiskReplay`;
- `RiskMetricsRecorder`;
- all stable structs/enums listed in this chapter.

Forbidden behavior:

- no implementation state hidden in globals;
- no side-effectful initialization;
- no metrics registration with global runtime clients from `lib.rs`.

Allowed dependencies:

- internal modules and workspace-local public domain types.

Forbidden dependencies:

- any external IO dependency.

Tests required:

- API compile tests for imports and feature-gated serialization.

### `src/engine.rs`

Responsibility:

- implement risk decision orchestration;
- route product/side-specific reservation paths;
- call duplicate order handler before any new reservation mutation;
- ensure event payloads are returned for append by the caller;
- reject deterministically on insufficient available balance, invalid quantity, overflow, or invariant failure.

Public items:

- `RiskEngine` trait;
- default engine struct such as `DefaultRiskEngine<S, B, C, M, R>` in implementation phase;
- `RiskDecision`;
- `RiskRejectReason`.

Forbidden behavior:

- no matching;
- no durable append;
- no ledger writes;
- no external service calls;
- no float math;
- no wall-clock decisions;
- no mutation before duplicate-order check.

Allowed dependencies:

- reservation, credit bucket, risk cache, margin, funding, liquidation, invariants, metrics, error modules.

Forbidden dependencies:

- IO, database, Kafka, Redis, async runtime, random number generation.

Tests required:

- accepted order creates exactly one reservation;
- duplicate accepted order returns existing reservation;
- rejected order has no active reservation;
- overflow rejects without mutation;
- Book Core boundary receives deterministic decision for same input and same state.

### `src/reservation.rs`

Responsibility:

- define `Reservation`, `ReservationId`, `ReservationState`, `ReservationDelta`, `ReservationConsume`, `ReservationRelease`;
- implement reservation state machine;
- enforce idempotent create, consume, and release semantics;
- track consumed and released amounts;
- produce event payload deltas for each mutation.

Public items:

- `ReservationStore` trait;
- `ReservationLifecycle` trait;
- reservation domain structs/enums.

Forbidden behavior:

- no negative amounts;
- no consume after terminal release beyond idempotent no-op for same operation key;
- no release after full consume except zero remaining idempotent release;
- no overwrite of reservation identity;
- no direct balance mutation without credit bucket mutation.

Allowed dependencies:

- fixed values, identifiers, domain order references, error types.

Forbidden dependencies:

- cold ledger clients;
- event log appenders;
- wall-clock time.

Tests required:

- create, active, partial consumed, fully consumed, released, rejected-release, cancel-release transitions;
- consume once;
- release once;
- consumed plus released never exceeds original reserved amount;
- duplicate idempotency keys are stable.

### `src/credit_bucket.rs`

Responsibility:

- model available, reserved, locked, and settled adjustment amounts per account/asset/bucket;
- apply reserve, consume, release, funding, liquidation, and ledger adjustment deltas;
- enforce nonnegative available, reserved, and locked;
- support bucket-level risk limits.

Public items:

- `CreditBucketStore` trait;
- `CreditBucket`;
- `CreditBucketId`.

Forbidden behavior:

- no total-changing mutation without explicit delta reason;
- no hidden overdraft;
- no unbounded bucket creation in hot path;
- no mutation that bypasses invariant validation.

Allowed dependencies:

- fixed amounts, ids, error, invariants.

Forbidden dependencies:

- IO adapters and network clients.

Tests required:

- reserve decreases available and increases reserved;
- consume decreases reserved and records settlement-facing delta;
- release decreases reserved and increases available;
- locked funds cannot be reserved;
- totals reconcile after sequences.

### `src/risk_cache.rs`

Responsibility:

- provide fast local lookup of user/asset risk state;
- map account/asset/symbol/product state to `UserRiskState`, `AssetRiskState`, and `RiskCacheEntry`;
- detect drift against reservation store, credit buckets, event projection, or cold ledger projection inputs.

Public items:

- `RiskCache` trait;
- `RiskCacheEntry`;
- `UserRiskState`;
- `AssetRiskState`.

Forbidden behavior:

- no cache-as-source-of-truth when event/replay state disagrees;
- no best-effort stale success;
- no lazy external IO lookup in hot path;
- no unbounded growth.

Allowed dependencies:

- fixed amounts, ids, error, invariants, metrics labels.

Forbidden dependencies:

- DB, Kafka, Redis, HTTP.

Tests required:

- lookup stable for same state;
- cache updates after reserve/consume/release;
- drift detection reports exact account/asset/bucket/reservation discrepancy.

### `src/margin.rs`

Responsibility:

- calculate futures margin requirement using fixed-point integers;
- validate reduce-only orders;
- calculate incremental margin for exposure-increasing orders;
- enforce nonnegative margin.

Public items:

- `MarginCalculator` trait;
- `MarginRequirement`.

Forbidden behavior:

- no floating point;
- no market data fetch;
- no dynamic fee/margin table fetch in hot path;
- no reduction order that increases exposure.

Allowed dependencies:

- fixed price/quantity/notional types, risk config snapshots, domain position views.

Forbidden dependencies:

- external pricing services;
- clocks;
- IO.

Tests required:

- initial margin calculation;
- maintenance margin check;
- reduce-only cannot increase exposure;
- overflow-safe notional multiplication;
- rounding is deterministic and conservative.

### `src/funding.rs`

Responsibility:

- apply replayable funding risk deltas to hot risk state;
- validate funding delta IDs and idempotency keys;
- adjust available/reserved/locked or settled adjustment fields according to approved event semantics.

Public items:

- `FundingRiskUpdater` trait;
- `FundingRiskDelta`.

Forbidden behavior:

- no funding rate calculation from floats;
- no external mark price read;
- no non-event-backed balance mutation.

Allowed dependencies:

- fixed signed amounts, ids, credit bucket and risk cache traits.

Forbidden dependencies:

- market data clients;
- ledgers;
- databases.

Tests required:

- positive funding credit;
- negative funding debit with insufficient available handled deterministically;
- duplicate funding delta is idempotent;
- replay reproduces funding-adjusted balances.

### `src/liquidation.rs`

Responsibility:

- apply replayable liquidation risk deltas;
- reserve or lock collateral according to liquidation event semantics;
- reduce positions and update hot margin risk state.

Public items:

- `LiquidationRiskUpdater` trait;
- `LiquidationRiskDelta`.

Forbidden behavior:

- no liquidation trigger decision based on external IO;
- no nondeterministic ordering between liquidation and user orders;
- no negative margin state.

Allowed dependencies:

- fixed amounts, ids, margin views, credit bucket and risk cache traits.

Forbidden dependencies:

- liquidation bots, market data adapters, external auction systems.

Tests required:

- liquidation delta reduces exposure;
- liquidation fee/debit is deterministic;
- duplicate liquidation delta is idempotent;
- invariant violation fails closed.

### `src/reconciliation.rs`

Responsibility:

- compare hot reservations, credit buckets, risk cache entries, event-derived projection, and cold ledger projection;
- produce `ReconciliationResult` without performing IO;
- define severity and recovery hints.

Public items:

- `RiskReconciler` trait;
- `ReconciliationResult`.

Forbidden behavior:

- no mutation during comparison unless explicitly named repair function is used in cold path;
- no hot-path external IO;
- no silent drift suppression.

Allowed dependencies:

- risk structs, event projection views, ledger projection views, errors, metrics labels.

Forbidden dependencies:

- database clients and filesystem traversal.

Tests required:

- exact match returns clean result;
- missing reservation detected;
- mismatched reserved amount detected;
- cold ledger lag handled as pending not success;
- unrecoverable divergence escalates.

### `src/config.rs`

Responsibility:

- define `RiskEngineConfig` and validation methods;
- hold fixed-point scales, fee caps, market order budget policy, max active reservations, bucket limits, replay validation flags, drift thresholds, and benchmark thresholds.

Public items:

- `RiskEngineConfig`;
- config validation helpers.

Forbidden behavior:

- no config loading from disk in hot module;
- no environment variable reads in deterministic processing;
- no default that disables invariant validation.

Allowed dependencies:

- fixed types and domain identifiers.

Forbidden dependencies:

- config file readers in core module;
- network config services.

Tests required:

- invalid scales reject;
- zero/negative limits reject;
- fee cap bounds validate;
- market buy budget policy validates.

### `src/error.rs`

Responsibility:

- define stable risk error, reject, and invariant violation types;
- map internal errors to external-safe reject reasons;
- avoid leaking sensitive account state in Display output.

Public items:

- `RiskEngineError`;
- `RiskRejectReason`;
- `RiskInvariantViolation`.

Forbidden behavior:

- no account balance secrets in user-facing messages;
- no unstable string-only error classification;
- no panic-based control flow.

Allowed dependencies:

- ids and fixed types if required for machine-readable diagnostics.

Forbidden dependencies:

- logging clients or telemetry backends.

Tests required:

- stable error codes;
- reject mapping;
- no sensitive debug output in Display.

### `src/metrics.rs`

Responsibility:

- define metrics recording trait and metric names;
- ensure metrics failures cannot alter risk decisions;
- keep metric labels bounded.

Public items:

- `RiskMetricsRecorder`;
- metric label enums.

Forbidden behavior:

- no global exporter requirement;
- no allocation-heavy label formatting in hot path;
- no blocking metrics flush in risk decision.

Allowed dependencies:

- error/reject enums and domain product/side labels.

Forbidden dependencies:

- Prometheus/OpenTelemetry concrete exporters in core crate unless feature-gated cold adapter.

Tests required:

- no-op recorder works;
- recorder errors are ignored or isolated according to trait contract;
- labels are finite and documented.

### `src/invariants.rs`

Responsibility:

- centralize invariant checks;
- validate bucket, reservation, cache, margin, and replay state;
- return structured violations.

Public items:

- invariant validation functions;
- `RiskInvariantViolation` if not fully owned by `error.rs`.

Forbidden behavior:

- no panic for user-caused invariant failures;
- no mutation;
- no clock or IO.

Allowed dependencies:

- risk structs and fixed types.

Forbidden dependencies:

- external clients.

Tests required:

- each invariant in the invariant table has at least one positive and one negative test.

### `src/replay.rs`

Responsibility:

- reconstruct risk state from `EngineEvent` views;
- apply reservation-created, consumed, released, funding-applied, liquidation-applied, and ledger-adjustment events deterministically;
- detect hash/sequence gaps when event metadata is provided by caller.

Public items:

- `RiskReplay` trait;
- replay state builder implementation in implementation phase.

Forbidden behavior:

- no event log IO;
- no snapshot file IO;
- no nondeterministic ordering;
- no external calls during replay.

Allowed dependencies:

- event view types, risk state modules, invariants.

Forbidden dependencies:

- storage engines and network clients.

Tests required:

- replay reservation lifecycle;
- replay duplicate retries;
- replay funding/liquidation deltas;
- replay equivalence after crash points;
- replay detects divergent terminal state.

## 9. Hot Risk vs Cold Risk Boundary

Hot risk is the synchronous, local decision path needed before an order enters Book Core.

Hot risk may:

- read local in-memory credit bucket state;
- read local in-memory reservation state;
- read local in-memory risk cache state;
- use a preloaded immutable configuration snapshot;
- calculate deterministic fixed-point amounts;
- return event payload data to the caller;
- update local state only when the caller's event append protocol permits the mutation sequence defined by the implementation.

Hot risk must not:

- call a database;
- call Kafka;
- call a ledger service;
- call a pricing service;
- wait on a network;
- read wall-clock time;
- allocate unbounded memory;
- block on a metrics exporter.

Cold risk is asynchronous audit, reconciliation, projection, and repair tooling. Cold risk may read event logs, ledger projections, snapshots, and reconciliation inputs, but it cannot retroactively change a hot decision. When cold risk finds drift, it must emit a structured reconciliation result and trigger the approved recovery workflow: halt, quarantine, replay, compare, and only then resume.

Boundary rule:

- The hot path answers: “Can this order safely enter matching right now?”
- The cold path answers: “Does durable, replayed, ledger-projected state agree with what hot risk believed?”

## 10. Reservation Lifecycle

Reservation states:

1. `PendingAppend` may be used internally when an event payload is prepared but not yet committed by the caller.
2. `Active` means the reservation protects an order that may match or rest.
3. `PartiallyConsumed` means at least one fill consumed part of the reservation, and remaining amount may still be active.
4. `FullyConsumed` means fills consumed the full reserved amount.
5. `Released` means remaining reservation was released because the order is no longer active.
6. `RejectedReleased` means a reservation-like predecision was unwound for a reject path; no active hold may remain.
7. `CancelledReleased` means cancel released all remaining hold exactly once.

Lifecycle transitions:

| From | Event | To | Rule |
| --- | --- | --- | --- |
| None | reserve accepted | Active | Duplicate key must not already map to a different order. |
| Active | trade consume partial | PartiallyConsumed | `consumed + released <= reserved_amount`. |
| Active | trade consume full | FullyConsumed | Remaining amount becomes zero. |
| PartiallyConsumed | trade consume remaining | FullyConsumed | Consume idempotency key required. |
| Active | cancel release | CancelledReleased | Release remaining exactly once. |
| PartiallyConsumed | cancel release | CancelledReleased | Release remaining exactly once. |
| Active | reject release | RejectedReleased | Only for pre-Book reject after reservation event protocol. |
| Any terminal | same operation key retry | same terminal | Idempotent no-op with same result. |
| Any terminal | conflicting operation | error | Fail closed; no mutation. |

Reservation lifecycle data must be serialized in `EngineEvent` payloads or derivable from them. Replay must never require a hidden in-memory-only flag.

## 11. Credit Bucket Model

A credit bucket is the smallest hot accounting container that can answer reservation availability.

Example fields:

- `bucket_id`;
- `account_id`;
- `asset_id`;
- `available`;
- `reserved`;
- `locked`;
- `settled_adjustments`;
- `active_reservation_count`;
- `version`.

Balance equation:

```text
total = available + reserved + locked + settled_adjustments
```

All values are fixed-point integers. If signed settled adjustments are used, validation must define whether `total` is signed or whether adjustments are normalized into explicit debit/credit fields.

Credit bucket mutation examples:

- reserve: `available -= amount`, `reserved += amount`;
- consume: `reserved -= amount`, settlement delta emitted for clearing;
- release: `reserved -= amount`, `available += amount`;
- lock: `available -= amount`, `locked += amount`;
- unlock: `locked -= amount`, `available += amount`;
- funding credit: `available += amount` or `settled_adjustments += amount` depending on event semantics;
- funding debit: `available -= amount` if available permits, otherwise deterministic lock/liquidation path if HES permits;
- liquidation: reserve/lock/release collateral according to liquidation event delta.

Bucket ownership:

- The hot single-writer owner mutates buckets.
- Readers receive immutable snapshots or copyable views.
- Cross-thread mutation must occur by deterministic command/event application, not shared mutable access.

## 12. Risk Cache Model

The risk cache is a performance view, not an accounting authority.

It may hold:

- per-user active reservation count;
- per-asset available/reserved/locked view;
- per-symbol position exposure;
- per-product margin summary;
- reduce-only exposure metadata;
- last applied event sequence;
- last reconciliation sequence.

Risk cache update rules:

- updates happen in the same deterministic order as reservation and bucket mutations;
- cache writes must be derived from authoritative local state transitions;
- cache drift is an invariant problem, not a warning to ignore;
- cache entries have bounded lifetime or bounded maximum count;
- cache lookups must not fetch missing data from cold storage in the hot path.

## 13. Spot Buy Reservation

Spot buy limit orders reserve quote asset.

Required amount:

```text
quote_notional = limit_price * base_quantity
fee_cap = max_taker_or_policy_fee(quote_notional, account_fee_tier, symbol_fee_policy)
reserve_amount = quote_notional + fee_cap
```

All multiplication must use fixed-point checked arithmetic with deterministic rounding. Rounding must be conservative: the reserved amount may be rounded up to avoid under-reserving, never down in a way that permits insufficient quote balance.

Acceptance checks:

- order quantity is positive;
- limit price is positive;
- notional multiplication does not overflow;
- fee calculation does not overflow;
- quote bucket exists or is explicitly preallocated;
- `available >= reserve_amount`;
- duplicate `client_order_id` does not already own a conflicting reservation.

Event data must include account, symbol, quote asset, reserved quote notional, fee cap, total reserve amount, reservation ID, client order ID, and deterministic arithmetic scale metadata if required by event schema.

## 14. Spot Sell Reservation

Spot sell limit orders reserve base asset.

Required amount:

```text
reserve_amount = base_quantity
```

If the venue charges fees in base asset before settlement, fee reserve must be added as a separate fee reservation component. If fees are charged in quote from proceeds, no additional base hold is needed for fees, but the event must identify the fee policy used.

Acceptance checks:

- quantity is positive;
- base bucket exists or is preallocated;
- `available >= base_quantity + base_fee_cap_if_applicable`;
- duplicate ID handling is complete before reserve mutation.

## 15. Market Buy Reservation

Market buy orders must be budget-bounded.

The risk engine must not infer an unlimited market notional from current book state unless HES explicitly allows a deterministic collar. The preferred implementation requires a client-provided quote budget or gateway-computed deterministic quote budget already validated by HES rules.

Required amount:

```text
reserve_amount = quote_budget + max_fee(quote_budget)
```

Rules:

- market buy with no budget is rejected;
- quote budget must be positive;
- fee reserve must use the maximum possible fee under policy;
- unused quote budget releases on completion/cancel/expiry;
- partial fills consume only actual quote spent plus actual fee; remaining budget remains reserved until terminal release.

## 16. Market Sell Reservation

Market sell orders reserve base quantity.

Required amount:

```text
reserve_amount = base_quantity + base_fee_cap_if_fee_asset_is_base
```

Rules:

- quantity must be positive;
- reserve full base quantity before matching;
- partial fills consume filled base quantity plus any base-denominated fee;
- unfilled base quantity releases at terminal state;
- market sell cannot create negative base available.

## 17. Fee Reservation

Fee reservation must be deterministic and conservative.

Fee policies must specify:

- fee asset;
- fee rate scale;
- maker/taker selection rule at reservation time;
- max fee cap rule;
- rebate handling rule;
- rounding direction;
- whether fee is reserved separately or included in quote/base amount;
- whether fee can be paid from proceeds.

Implementation guidance:

- Reserve maximum plausible fee before matching when the fee may debit current available balance.
- Do not use floating-point percentages; use integer basis points, parts-per-million, or fixed-point ratio types.
- If maker fee can be lower than taker fee, reserve taker fee unless HES explicitly permits a smaller deterministic cap.
- Rebates must not increase available balance in hot risk until represented by replayable event data.
- Fee release occurs after actual fee is known and consumed; unneeded fee cap is released exactly once.

## 18. Futures Margin Reservation

Futures margin reservation applies to orders that can increase exposure.

Required information:

- account ID;
- symbol ID;
- side;
- price or market budget/collar;
- quantity;
- current position;
- existing open-order exposure;
- collateral asset;
- initial margin rate;
- maintenance margin rate;
- fee cap;
- leverage limit;
- reduce-only flag.

Required amount:

```text
new_exposure = current_position + open_order_exposure + order_exposure
incremental_initial_margin = max(0, initial_margin(new_exposure) - initial_margin(current_position + open_order_exposure))
reserve_amount = incremental_initial_margin + fee_cap
```

Rules:

- margin cannot be negative;
- notional multiplication must be checked;
- rounding must be conservative;
- reduce-only orders must not increase exposure;
- margin deltas are replayable;
- funding and liquidation deltas must update available margin deterministically.

## 19. Reduce-Only Risk Handling

Reduce-only order acceptance must prove the order cannot increase absolute exposure.

Required checks:

- account has an existing position on the symbol;
- order side opposes the position direction;
- order quantity is less than or equal to reducible quantity after considering earlier reduce-only open orders in deterministic order;
- the order cannot flip the position if fully filled;
- if partial fill plus later cancel occurs, remaining reduce-only hold releases deterministically;
- duplicate reduce-only orders do not double count reducible quantity.

If a reduce-only order fails these checks, reject with `RiskRejectReason::ReduceOnlyWouldIncreaseExposure` or equivalent stable code.

## 20. Funding Risk Updates

Funding updates are risk state mutations caused by replayable funding events.

Rules:

- funding delta must carry deterministic event identity;
- applying the same funding delta twice is idempotent;
- positive funding credit must not depend on wall-clock time;
- negative funding debit must not create hidden negative available balance;
- if debit cannot be applied to available balance, the configured deterministic handling path must be used: lock collateral, mark account under margin stress, or reject subsequent exposure-increasing orders;
- replay applies funding deltas in event order and reconstructs identical bucket state.

Funding updates must never be calculated from floats in `hermes-risk`. If rates come from another module, they must arrive as fixed-point integers in event data or an immutable risk input snapshot.

## 21. Liquidation Risk Updates

Liquidation updates are risk state mutations caused by liquidation events.

Rules:

- liquidation delta must identify account, symbol, position reduction, collateral debit/credit, fee, and reservation effects;
- duplicate liquidation delta is idempotent;
- liquidation may lock collateral, consume margin reservation, release stale reservations, or reduce exposure according to event data;
- liquidation cannot leave negative margin;
- liquidation cannot silently erase active reservations; it must emit reservation release/consume event data or receive such event data from the liquidation pipeline;
- replay applies liquidation deltas in event order.

## 22. Hold Consume

A hold is consumed when a trade or settlement-facing execution uses part of a reservation.

Consume rules:

- consume references `reservation_id` and `operation_id`;
- operation ID is idempotency key for retry;
- amount must be positive unless zero-amount idempotent replay is explicitly represented;
- amount cannot exceed remaining reserved amount;
- consumed amount reduces bucket `reserved`;
- actual settlement debit/credit must be represented in event data;
- after full consume, reservation becomes `FullyConsumed`;
- after partial consume, reservation remains active or partially consumed until terminal release/consume.

## 23. Hold Release

A hold is released when reserved funds are no longer required.

Release rules:

- release references `reservation_id` and `operation_id`;
- release amount is normally remaining amount;
- same release operation replay returns same result;
- release cannot exceed remaining amount;
- released amount reduces bucket `reserved` and increases `available`;
- terminal release state records reason;
- release event data includes reason, amount, remaining before/after, and linked order.

## 24. Reject Release

A rejected order normally should not create an active reservation. If implementation uses a staged reservation before final Book Core validation, rejection must unwind with `RejectedReleased` event data before any external reject response.

Rules:

- no active reservation after reject;
- release exactly once;
- duplicate reject retry returns existing reject decision and release outcome;
- reject release must be auditable;
- reject release must not depend on a cold ledger call.

## 25. Cancel Release

Cancel release occurs when an active or partially filled order is cancelled, expired, or otherwise removed from the book.

Rules:

- release remaining amount, not original amount;
- if fully consumed, release amount is zero and terminal state remains deterministic;
- cancellation retry is idempotent;
- cancel release links to cancel event ID or cancel operation ID;
- partial fill followed by cancel must satisfy `consumed + released = reserved_amount` unless fees/proceeds split requires component-level accounting.

## 26. Duplicate Order / Idempotency Handling

Duplicate `client_order_id` handling happens before any new reservation mutation.

Identity key:

```text
(account_id, client_order_id, symbol_id or scope, product_type)
```

Rules:

- exact duplicate of an accepted order returns the existing `RiskDecision::Accepted` and reservation ID;
- exact duplicate of a rejected order returns the existing reject result if reject decision was event-backed or cached under approved idempotency policy;
- duplicate with conflicting order parameters rejects with `RiskRejectReason::DuplicateClientOrderIdConflict`;
- duplicate retry after unknown gateway response must not double reserve;
- idempotency records must be replayable from events or reconstructed from deterministic event payloads;
- idempotency map must be bounded and scoped by HES retention rules.

## 27. Reservation Reconciliation

Reservation reconciliation compares:

- active reservation store;
- credit bucket reserved totals;
- risk cache reserved totals;
- event-derived reservation projection;
- order state from Book Core projections if supplied.

Required checks:

- every active order has exactly one active or partially consumed reservation;
- every active reservation maps to an order that is active or pending terminal event;
- bucket reserved amount equals sum of reservation remaining amounts by account/asset/bucket;
- terminal reservations are not included in active reserved totals;
- idempotency records point to the correct reservation or reject result.

Outcomes:

- `Clean`;
- `PendingColdProjection`;
- `DriftDetected`;
- `InvariantViolation`;
- `ReplayRequired`;
- `QuarantineRequired`.

## 28. Cold Ledger Reconciliation

Cold ledger reconciliation compares event/replay-derived hot risk state with ledger projection.

Rules:

- cold ledger projection may lag; lag is not automatically drift;
- reconciliation input must include ledger projection sequence or event watermark;
- compare only states at compatible sequence boundaries;
- unresolved discrepancy after watermark alignment is drift;
- drift triggers halt/quarantine/replay workflow according to operations policy;
- hot path must not wait for cold ledger reconciliation.

Comparison examples:

- account/asset available from replayed risk state equals ledger-projected available after all settlement events through watermark;
- reserved totals from event projection equal hot reservation store at same sequence;
- funding and liquidation adjustments are present in both projections;
- cancelled/rejected orders leave no active holds.

## 29. Risk Cache Drift Detection

Risk cache drift detection verifies cache entries against authoritative in-memory stores or replay projections.

Drift categories:

- available mismatch;
- reserved mismatch;
- locked mismatch;
- active reservation count mismatch;
- idempotency mapping mismatch;
- margin requirement mismatch;
- event sequence mismatch.

Severity:

- cache-only drift with correct underlying state: rebuild cache before continuing if hot path relies on cache;
- underlying store drift: halt and replay;
- event sequence drift: quarantine until hash-chain verification completes.

## 30. Failure Handling

Failure handling must be deterministic and fail closed.

Rules:

- validation failure before mutation returns reject;
- arithmetic overflow returns reject or internal error according to context, with no mutation;
- invariant violation after mutation requires halt/quarantine because state may be corrupt;
- event append failure after prepared mutation must not publish success;
- if implementation mutates before append for latency reasons, it must have an approved rollback or crash-replay protocol; otherwise, use event append-before-visible-success sequence;
- duplicate retry after a crash must reconstruct reservation state from events and avoid double reserve;
- metrics/logging failures are non-fatal and cannot alter decision;
- serialization failure for event payload is fatal before mutation or append.

## 31. Replay Reconstruction

Replay reconstructs:

- credit buckets;
- reservations;
- idempotency map;
- risk cache or the source state needed to rebuild it;
- funding and liquidation applied-delta sets;
- margin exposure summaries;
- last applied sequence and hash-chain head if supplied.

Replay rules:

- apply events in durable sequence order;
- reject sequence gaps;
- reject hash-chain mismatch when hash data is provided;
- do not use wall-clock time;
- do not call external services;
- validate invariants after each event in strict mode or at configured intervals in benchmark mode;
- final state must equal live state for the same event prefix.

## 32. Performance Considerations

Hot path performance goals should be defined by benchmark gates, but implementation should be shaped by these constraints:

- O(1) average reservation lookup by reservation ID;
- O(1) average idempotency lookup by duplicate key;
- O(1) bucket lookup for preallocated buckets;
- no heap allocation on the common accepted/rejected path after warmup where feasible;
- bounded maps with explicit capacity;
- deterministic eviction or no hot-path eviction;
- checked arithmetic with minimal branching;
- stable memory layout for cache-friendly bucket and reservation structs;
- no trait object allocation in critical loops unless benchmarked and approved;
- metrics updates should be lock-free or no-op in hot profiles.

## 33. Observability

Observability must explain risk decisions without changing them.

Required observability artifacts:

- counters for accepts and rejects by reason;
- counters for reservation create/consume/release;
- counters for duplicate order idempotent returns and conflicts;
- gauges for active reservations, reserved total by asset, locked total by asset, and drift count;
- histograms for reserve, consume, release, and cache lookup latency;
- structured logs for invariant violations, reconciliation drift, replay divergence, and quarantine triggers;
- event sequence and reservation ID correlation fields.

Sensitive data rule:

- logs may include stable IDs and reason codes;
- logs must not include secrets, private keys, raw auth headers, or unnecessary full balances in user-facing contexts;
- operational logs may include machine-readable deltas for audit if access-controlled.

## 34. Metrics

Metric names should be finite and documented.

Suggested metrics:

- `risk_order_decisions_total{product,side,decision,reason}`;
- `risk_reservations_created_total{product,side,asset}`;
- `risk_reservations_consumed_total{asset}`;
- `risk_reservations_released_total{reason,asset}`;
- `risk_duplicate_client_order_total{outcome}`;
- `risk_invariant_violations_total{kind}`;
- `risk_reconciliation_runs_total{outcome}`;
- `risk_cache_drift_total{kind}`;
- `risk_replay_events_applied_total{event_type}`;
- `risk_reserve_latency_ns`;
- `risk_consume_latency_ns`;
- `risk_release_latency_ns`;
- `risk_cache_lookup_latency_ns`;
- `risk_active_reservations`;
- `risk_reserved_amount_units{asset}`.

Labels must be bounded. Never label by raw account ID, order ID, reservation ID, or client order ID in high-cardinality hot metrics.

## 35. Logging

Log levels:

- `TRACE`: disabled by default; may include local state transition names in development.
- `DEBUG`: deterministic branch and benchmark diagnostics in non-production.
- `INFO`: startup config summary, replay completion, reconciliation clean summary.
- `WARN`: cold projection lag, cache rebuild, duplicate conflict, recoverable drift.
- `ERROR`: invariant violation, replay divergence, append protocol failure, unrecoverable reconciliation drift.

Required structured fields:

- `component = "hermes-risk"`;
- `event_sequence` when known;
- `account_id_hash` or stable redacted account identifier if approved;
- `symbol_id`;
- `asset_id`;
- `reservation_id` when relevant;
- `client_order_id_hash` when relevant;
- `reason_code`;
- `operation_id` for consume/release idempotency.

## 36. Configuration

`RiskEngineConfig` must be fully validated before use.

Configuration fields:

- `max_active_reservations_per_account`;
- `max_active_reservations_total`;
- `max_credit_buckets`;
- `max_idempotency_records`;
- `fixed_price_scale`;
- `fixed_quantity_scale`;
- `fixed_amount_scale`;
- `fee_rate_scale`;
- `default_taker_fee_rate`;
- `default_maker_fee_rate`;
- `market_buy_requires_quote_budget`;
- `market_order_fee_cap_multiplier`;
- `futures_initial_margin_rate_by_symbol`;
- `futures_maintenance_margin_rate_by_symbol`;
- `reduce_only_strict_mode`;
- `replay_validate_every_event`;
- `reconciliation_max_projection_lag_events`;
- `halt_on_cache_drift`;
- `metrics_enabled`.

Validation:

- all capacities must be positive and bounded;
- scales must match fixed type definitions;
- fee rates must be nonnegative;
- initial margin must be greater than or equal to maintenance margin;
- market buy budget must be required unless an approved deterministic collar exists;
- no config may enable floats.

## 37. Error Types

Stable error categories:

- `InvalidOrder`;
- `DuplicateClientOrderIdConflict`;
- `InsufficientAvailable`;
- `InsufficientMargin`;
- `ReduceOnlyWouldIncreaseExposure`;
- `ReservationNotFound`;
- `ReservationAlreadyTerminal`;
- `ReservationOperationConflict`;
- `ArithmeticOverflow`;
- `InvariantViolation`;
- `CacheDrift`;
- `ReplayDiverged`;
- `ColdLedgerDrift`;
- `ConfigurationInvalid`;
- `CapacityExceeded`;
- `UnsupportedProduct`;
- `UnsupportedFeePolicy`.

Error behavior:

- user-facing reject reasons must be stable and serializable;
- internal errors must include machine-readable codes;
- no error path may panic for expected invalid input;
- invariant violations are not normal rejects and should trigger fail-closed workflow;
- errors returned before mutation must guarantee no state change;
- errors returned after detecting corrupt state must mark state unsafe for continued trading.

## 38. Public Traits

### `RiskEngine`

Purpose:

- orchestrate order risk checks and reservation creation before Book Core matching.

Rust-style contract:

```rust
pub trait RiskEngine {
    type Order;
    type EventDelta;

    fn reserve_for_order(&mut self, order: &Self::Order) -> Result<RiskDecision<Self::EventDelta>, RiskEngineError>;

    fn consume_reservation_for_trade(
        &mut self,
        consume: ReservationConsume,
    ) -> Result<ReservationDelta, RiskEngineError>;

    fn release_reservation_for_cancel(
        &mut self,
        release: ReservationRelease,
    ) -> Result<ReservationDelta, RiskEngineError>;

    fn release_reservation_for_reject(
        &mut self,
        release: ReservationRelease,
    ) -> Result<ReservationDelta, RiskEngineError>;

    fn validate_invariants(&self) -> Result<(), RiskInvariantViolation>;
}
```

Ownership model:

- mutable single-writer ownership for hot state;
- immutable borrowed order input;
- returned decision owns event delta payloads for caller append.

Error behavior:

- deterministic rejects use `RiskDecision::Rejected`;
- corrupt internal state uses `RiskEngineError::InvariantViolation`;
- arithmetic overflow returns `RiskEngineError::ArithmeticOverflow` or stable reject according to operation phase.

Determinism requirements:

- no wall-clock, random, external IO, or float operations;
- same input state and order produce same decision and deltas.

Idempotency requirements:

- duplicate `client_order_id` returns existing decision;
- consume/release operation IDs are idempotent.

Test requirements:

- duplicate retry;
- accepted reservation;
- reject no mutation;
- trade consume;
- cancel release;
- replay equivalence.

### `ReservationStore`

Purpose:

- provide deterministic storage and lookup for reservations and idempotency mappings.

```rust
pub trait ReservationStore {
    fn get(&self, id: ReservationId) -> Option<&Reservation>;
    fn get_mut(&mut self, id: ReservationId) -> Option<&mut Reservation>;
    fn find_by_client_order(&self, key: ClientOrderKey) -> Option<ReservationId>;
    fn insert_active(&mut self, reservation: Reservation) -> Result<(), RiskEngineError>;
    fn record_client_order_mapping(&mut self, key: ClientOrderKey, id: ReservationId) -> Result<(), RiskEngineError>;
    fn active_reservations_for_bucket(&self, bucket: CreditBucketId) -> ReservationIterator<'_>;
}
```

Ownership model:

- store is mutably owned by single writer;
- immutable iterators cannot mutate reservations.

Error behavior:

- capacity overflow returns `CapacityExceeded`;
- conflicting mapping returns `DuplicateClientOrderIdConflict`.

Determinism requirements:

- iteration order for replay/reconciliation must be stable.

Idempotency requirements:

- existing mapping cannot be overwritten by a different order.

Test requirements:

- insert/get;
- duplicate mapping;
- stable iteration;
- capacity limit.

### `ReservationLifecycle`

Purpose:

- apply state transitions to a reservation.

```rust
pub trait ReservationLifecycle {
    fn consume(&mut self, consume: ReservationConsume) -> Result<ReservationDelta, RiskEngineError>;
    fn release(&mut self, release: ReservationRelease) -> Result<ReservationDelta, RiskEngineError>;
    fn remaining_amount(&self) -> FixedAmount;
    fn is_terminal(&self) -> bool;
}
```

Ownership model:

- mutation requires `&mut self` reservation ownership.

Error behavior:

- conflicting operation IDs return `ReservationOperationConflict`;
- over-consume/over-release returns invariant violation or operation error.

Determinism requirements:

- state transition depends only on reservation fields and operation input.

Idempotency requirements:

- same operation ID and same parameters returns same delta/no-op result.

Test requirements:

- consume once;
- release once;
- conflict detection;
- terminal state behavior.

### `CreditBucketStore`

Purpose:

- mutate and query bucket-level available/reserved/locked state.

```rust
pub trait CreditBucketStore {
    fn bucket(&self, id: CreditBucketId) -> Option<&CreditBucket>;
    fn bucket_mut(&mut self, id: CreditBucketId) -> Option<&mut CreditBucket>;
    fn reserve(&mut self, id: CreditBucketId, amount: FixedAmount) -> Result<(), RiskEngineError>;
    fn consume_reserved(&mut self, id: CreditBucketId, amount: FixedAmount) -> Result<(), RiskEngineError>;
    fn release_reserved(&mut self, id: CreditBucketId, amount: FixedAmount) -> Result<(), RiskEngineError>;
    fn apply_settled_adjustment(&mut self, id: CreditBucketId, amount: SignedFixedAmount) -> Result<(), RiskEngineError>;
    fn validate_bucket(&self, id: CreditBucketId) -> Result<(), RiskInvariantViolation>;
}
```

Ownership model:

- one mutable bucket operation at a time under the hot single writer.

Error behavior:

- insufficient available returns `InsufficientAvailable`;
- underflow/overflow returns arithmetic error;
- invariant failure fails closed.

Determinism requirements:

- fixed order of operations and checked integer arithmetic.

Idempotency requirements:

- store methods are not individually idempotent; callers must use reservation/funding/liquidation operation IDs.

Test requirements:

- reserve/consume/release totals;
- insufficient funds;
- lock interaction;
- settled adjustment.

### `RiskCache`

Purpose:

- provide bounded low-latency risk views and cache drift detection.

```rust
pub trait RiskCache {
    fn user_state(&self, account_id: AccountId) -> Option<&UserRiskState>;
    fn asset_state(&self, account_id: AccountId, asset_id: AssetId) -> Option<&AssetRiskState>;
    fn update_after_reservation(&mut self, delta: &ReservationDelta) -> Result<(), RiskEngineError>;
    fn update_after_funding(&mut self, delta: &FundingRiskDelta) -> Result<(), RiskEngineError>;
    fn update_after_liquidation(&mut self, delta: &LiquidationRiskDelta) -> Result<(), RiskEngineError>;
    fn detect_drift(&self, source: &AuthoritativeRiskView) -> Result<(), RiskInvariantViolation>;
}
```

Ownership model:

- mutable updates follow authoritative store mutations.

Error behavior:

- capacity problems return `CapacityExceeded`;
- drift returns `RiskInvariantViolation`.

Determinism requirements:

- cache state is rebuildable from source state and events.

Idempotency requirements:

- duplicate deltas must be recognized by upstream operation IDs or applied via replay idempotency sets.

Test requirements:

- update after reserve/consume/release;
- rebuild equivalence;
- drift detection.

### `MarginCalculator`

Purpose:

- calculate deterministic futures margin requirements.

```rust
pub trait MarginCalculator {
    fn initial_margin(&self, input: MarginInput) -> Result<MarginRequirement, RiskEngineError>;
    fn maintenance_margin(&self, input: MarginInput) -> Result<MarginRequirement, RiskEngineError>;
    fn incremental_margin_for_order(&self, input: OrderMarginInput) -> Result<MarginRequirement, RiskEngineError>;
    fn validate_reduce_only(&self, input: ReduceOnlyInput) -> Result<(), RiskRejectReason>;
}
```

Ownership model:

- calculator should be stateless or hold immutable config snapshot.

Error behavior:

- overflow returns arithmetic error;
- reduce-only failure returns stable reject reason.

Determinism requirements:

- no floats; conservative fixed rounding.

Idempotency requirements:

- pure calculations; idempotency handled by caller.

Test requirements:

- initial/maintenance/incremental calculations;
- reduce-only edge cases;
- rounding vectors.

### `FundingRiskUpdater`

Purpose:

- apply event-backed funding deltas to hot risk state.

```rust
pub trait FundingRiskUpdater {
    fn apply_funding_risk_update(&mut self, delta: FundingRiskDelta) -> Result<ReservationDelta, RiskEngineError>;
    fn has_applied_funding_delta(&self, funding_event_id: EventId) -> bool;
}
```

Ownership model:

- mutable hot state owner applies deltas in event order.

Error behavior:

- duplicate returns prior result or idempotent no-op;
- invalid debit path returns invariant/configuration error.

Determinism requirements:

- delta already contains fixed amount; no rate/time calculation.

Idempotency requirements:

- event ID must prevent double apply.

Test requirements:

- credit/debit/duplicate/replay.

### `LiquidationRiskUpdater`

Purpose:

- apply event-backed liquidation deltas to hot risk state.

```rust
pub trait LiquidationRiskUpdater {
    fn apply_liquidation_risk_update(&mut self, delta: LiquidationRiskDelta) -> Result<ReservationDelta, RiskEngineError>;
    fn has_applied_liquidation_delta(&self, liquidation_event_id: EventId) -> bool;
}
```

Ownership model:

- mutable hot state owner applies liquidation in deterministic event order.

Error behavior:

- duplicate is idempotent;
- negative margin or missing bucket fails closed.

Determinism requirements:

- no external trigger calculation; apply supplied delta only.

Idempotency requirements:

- event ID prevents double liquidation adjustment.

Test requirements:

- duplicate, exposure reduction, collateral debit, invariant failure.

### `RiskReconciler`

Purpose:

- compare hot risk state with event and ledger projections.

```rust
pub trait RiskReconciler {
    fn reconcile_reservations(&self, input: ReservationReconciliationInput<'_>) -> ReconciliationResult;
    fn reconcile_with_cold_ledger(&self, input: ColdLedgerReconciliationInput<'_>) -> ReconciliationResult;
    fn detect_risk_cache_drift(&self, input: CacheDriftInput<'_>) -> ReconciliationResult;
}
```

Ownership model:

- immutable comparison; repair is outside this trait unless explicitly added as cold-only API.

Error behavior:

- returns structured result, not panic.

Determinism requirements:

- stable sort/order of discrepancies.

Idempotency requirements:

- repeated reconciliation over same inputs returns identical result.

Test requirements:

- clean, lagged, drift, invariant violation cases.

### `RiskReplay`

Purpose:

- rebuild hot risk state from event stream.

```rust
pub trait RiskReplay {
    type State;
    type EventView;

    fn empty_state(config: RiskEngineConfig) -> Self::State;
    fn apply_event(state: &mut Self::State, event: &Self::EventView) -> Result<(), RiskEngineError>;
    fn finalize(state: Self::State) -> Result<Self::State, RiskEngineError>;
}
```

Ownership model:

- replay owns mutable reconstruction state.

Error behavior:

- sequence/hash mismatch or invariant failure returns replay error.

Determinism requirements:

- event order only; no external reads.

Idempotency requirements:

- replay may accept duplicate idempotent operation events only if event schema requires it; otherwise duplicates are divergence.

Test requirements:

- lifecycle replay, crash points, cold ledger reconciliation after replay.

### `RiskMetricsRecorder`

Purpose:

- isolate metrics from risk decisions.

```rust
pub trait RiskMetricsRecorder {
    fn record_decision(&self, product: ProductType, side: Side, decision: RiskDecisionLabel, reason: Option<RiskRejectReason>);
    fn record_reservation_created(&self, asset_id: AssetId, amount: FixedAmount);
    fn record_reservation_consumed(&self, asset_id: AssetId, amount: FixedAmount);
    fn record_reservation_released(&self, asset_id: AssetId, amount: FixedAmount, reason: ReleaseReason);
    fn record_latency_ns(&self, operation: RiskOperation, elapsed_ns: u64);
    fn record_invariant_violation(&self, violation: &RiskInvariantViolation);
}
```

Ownership model:

- shared immutable recorder reference; implementation must be nonblocking or no-op.

Error behavior:

- trait intentionally returns `()` so metrics failures cannot affect decisions.

Determinism requirements:

- metrics may observe but not influence state.

Idempotency requirements:

- metrics can count retries; business idempotency does not depend on metrics.

Test requirements:

- no-op recorder; bounded labels; recorder panic policy if workspace permits catch-free implementations.

## 39. Core Structs and Enums

### `RiskEngineConfig`

```rust
pub struct RiskEngineConfig {
    pub max_active_reservations_per_account: u32,
    pub max_active_reservations_total: u64,
    pub max_credit_buckets: u32,
    pub max_idempotency_records: u64,
    pub fixed_price_scale: u32,
    pub fixed_quantity_scale: u32,
    pub fixed_amount_scale: u32,
    pub fee_rate_scale: u32,
    pub default_taker_fee_rate: FixedRate,
    pub default_maker_fee_rate: FixedRate,
    pub market_buy_requires_quote_budget: bool,
    pub market_order_fee_cap_multiplier: FixedRate,
    pub reduce_only_strict_mode: bool,
    pub replay_validate_every_event: bool,
    pub reconciliation_max_projection_lag_events: u64,
    pub halt_on_cache_drift: bool,
}
```

Purpose: immutable validated risk configuration.

Invariants: capacities positive; scales match fixed types; fee rates nonnegative; market buy budget required unless approved collar exists.

Ownership rules: clone/copy snapshot into hot engine at startup; no runtime mutation in hot path.

Serialization: may serialize for config audit; event payloads should store config version, not full config.

### `RiskDecision`

```rust
pub enum RiskDecision<E> {
    Accepted { reservation_id: ReservationId, event_delta: E },
    Rejected { reason: RiskRejectReason, event_delta: Option<E> },
    DuplicateAccepted { reservation_id: ReservationId, original_event_sequence: EventSequence },
    DuplicateRejected { reason: RiskRejectReason, original_event_sequence: Option<EventSequence> },
}
```

Purpose: deterministic result of risk admission.

Invariants: accepted variants always identify existing reservation; rejected variants leave no active reservation.

Ownership rules: owns event delta to append or references original sequence for duplicate.

Serialization: reject reason and reservation ID must be stable in event/API schemas.

### `RiskRejectReason`

```rust
pub enum RiskRejectReason {
    InvalidQuantity,
    InvalidPrice,
    MissingMarketBuyBudget,
    InsufficientAvailable,
    InsufficientMargin,
    DuplicateClientOrderIdConflict,
    ReduceOnlyWouldIncreaseExposure,
    UnsupportedProduct,
    UnsupportedFeePolicy,
    CapacityExceeded,
    ArithmeticOverflow,
}
```

Purpose: user-safe stable reject reason.

Invariants: no sensitive balances embedded.

Ownership rules: copyable enum.

Serialization: stable numeric/string codes; never rename without versioning.

### `Reservation`

```rust
pub struct Reservation {
    pub id: ReservationId,
    pub account_id: AccountId,
    pub client_order_id: ClientOrderId,
    pub order_id: Option<OrderId>,
    pub symbol_id: SymbolId,
    pub product_type: ProductType,
    pub side: Side,
    pub bucket_id: CreditBucketId,
    pub asset_id: AssetId,
    pub reserved_amount: FixedAmount,
    pub consumed_amount: FixedAmount,
    pub released_amount: FixedAmount,
    pub fee_reserved_amount: FixedAmount,
    pub state: ReservationState,
    pub create_operation_id: OperationId,
    pub last_operation_id: Option<OperationId>,
    pub create_event_sequence: Option<EventSequence>,
}
```

Purpose: one auditable hold for an order or order component.

Invariants: reserved, consumed, released, and fee fields nonnegative; consumed plus released less than or equal to reserved; terminal state has no active remaining amount.

Ownership rules: mutable only by reservation lifecycle under single-writer risk owner.

Serialization: event payloads must contain enough fields to reconstruct it or derive it.

### `ReservationId`

```rust
pub struct ReservationId(pub u128);
```

Purpose: stable reservation identity.

Invariants: deterministic generation from event/order sequence or assigned by approved ID module; never random in replay.

Ownership rules: copyable value.

Serialization: fixed-width canonical encoding.

### `ReservationState`

```rust
pub enum ReservationState {
    Active,
    PartiallyConsumed,
    FullyConsumed,
    Released,
    RejectedReleased,
    CancelledReleased,
}
```

Purpose: state machine status.

Invariants: terminal variants have deterministic remaining amount semantics.

Ownership rules: changed only through lifecycle functions.

Serialization: stable codes.

### `ReservationDelta`

```rust
pub struct ReservationDelta {
    pub reservation_id: ReservationId,
    pub bucket_id: CreditBucketId,
    pub asset_id: AssetId,
    pub amount: FixedAmount,
    pub before_state: ReservationState,
    pub after_state: ReservationState,
    pub operation_id: OperationId,
    pub reason: ReservationDeltaReason,
}
```

Purpose: event-ready mutation record.

Invariants: amount nonnegative; before/after transition valid.

Ownership rules: returned by lifecycle and consumed by caller/event builder.

Serialization: must be event-schema stable.

### `ReservationConsume`

```rust
pub struct ReservationConsume {
    pub reservation_id: ReservationId,
    pub operation_id: OperationId,
    pub trade_id: TradeId,
    pub amount: FixedAmount,
    pub fee_amount: FixedAmount,
    pub event_sequence: Option<EventSequence>,
}
```

Purpose: idempotent consume command for fill/trade.

Invariants: amount and fee nonnegative; operation ID unique per consume.

Ownership rules: passed by value or immutable reference to lifecycle.

Serialization: consume event must include operation ID and trade ID.

### `ReservationRelease`

```rust
pub struct ReservationRelease {
    pub reservation_id: ReservationId,
    pub operation_id: OperationId,
    pub reason: ReleaseReason,
    pub requested_amount: Option<FixedAmount>,
    pub event_sequence: Option<EventSequence>,
}
```

Purpose: idempotent release command for cancel/reject/expiry.

Invariants: requested amount, if present, nonnegative and not above remaining.

Ownership rules: passed by value or immutable reference.

Serialization: release event must include reason and operation ID.

### `CreditBucket`

```rust
pub struct CreditBucket {
    pub id: CreditBucketId,
    pub account_id: AccountId,
    pub asset_id: AssetId,
    pub available: FixedAmount,
    pub reserved: FixedAmount,
    pub locked: FixedAmount,
    pub settled_adjustments: SignedFixedAmount,
    pub active_reservation_count: u32,
    pub version: u64,
}
```

Purpose: local balance container for hot risk.

Invariants: available/reserved/locked nonnegative; total equation holds; version monotonic.

Ownership rules: single-writer mutable.

Serialization: replay snapshots may serialize; events serialize deltas.

### `CreditBucketId`

```rust
pub struct CreditBucketId(pub u128);
```

Purpose: stable key for account/asset/risk bucket.

Invariants: deterministic mapping for account and asset unless HES specifies sub-buckets.

Ownership rules: copyable.

Serialization: fixed-width canonical encoding.

### `RiskCacheEntry`

```rust
pub struct RiskCacheEntry {
    pub account_id: AccountId,
    pub asset_id: AssetId,
    pub available: FixedAmount,
    pub reserved: FixedAmount,
    pub locked: FixedAmount,
    pub active_reservation_count: u32,
    pub last_event_sequence: EventSequence,
}
```

Purpose: query-optimized view.

Invariants: mirrors source state at sequence.

Ownership rules: rebuildable; not authoritative.

Serialization: cold snapshots only, not required for events if rebuildable.

### `UserRiskState`

```rust
pub struct UserRiskState {
    pub account_id: AccountId,
    pub assets: BoundedAssetMap<AssetId, AssetRiskState>,
    pub active_reservations: u32,
    pub last_event_sequence: EventSequence,
}
```

Purpose: aggregate account risk view.

Invariants: asset sums equal underlying buckets.

Ownership rules: cache-owned view.

Serialization: optional snapshot.

### `AssetRiskState`

```rust
pub struct AssetRiskState {
    pub asset_id: AssetId,
    pub bucket_id: CreditBucketId,
    pub available: FixedAmount,
    pub reserved: FixedAmount,
    pub locked: FixedAmount,
    pub settled_adjustments: SignedFixedAmount,
}
```

Purpose: per-asset risk view.

Invariants: nonnegative available/reserved/locked.

Ownership rules: derived from bucket.

Serialization: optional snapshot.

### `MarginRequirement`

```rust
pub struct MarginRequirement {
    pub symbol_id: SymbolId,
    pub collateral_asset_id: AssetId,
    pub initial_margin: FixedAmount,
    pub maintenance_margin: FixedAmount,
    pub fee_cap: FixedAmount,
    pub incremental_initial_margin: FixedAmount,
}
```

Purpose: deterministic margin result for futures orders.

Invariants: all fields nonnegative; initial >= maintenance; incremental nonnegative.

Ownership rules: pure calculation output.

Serialization: include in event or derivable from event inputs and config version.

### `FundingRiskDelta`

```rust
pub struct FundingRiskDelta {
    pub event_id: EventId,
    pub account_id: AccountId,
    pub symbol_id: SymbolId,
    pub collateral_asset_id: AssetId,
    pub amount: SignedFixedAmount,
    pub bucket_id: CreditBucketId,
    pub operation_id: OperationId,
}
```

Purpose: replayable funding adjustment.

Invariants: event ID unique; amount fixed-point signed integer.

Ownership rules: applied by funding updater.

Serialization: required in funding event.

### `LiquidationRiskDelta`

```rust
pub struct LiquidationRiskDelta {
    pub event_id: EventId,
    pub account_id: AccountId,
    pub symbol_id: SymbolId,
    pub collateral_asset_id: AssetId,
    pub collateral_delta: SignedFixedAmount,
    pub position_reduction_qty: FixedQuantity,
    pub fee_amount: FixedAmount,
    pub bucket_id: CreditBucketId,
    pub operation_id: OperationId,
}
```

Purpose: replayable liquidation adjustment.

Invariants: position reduction and fee nonnegative; collateral delta fixed signed amount; cannot create negative margin.

Ownership rules: applied by liquidation updater.

Serialization: required in liquidation event.

### `ReconciliationResult`

```rust
pub enum ReconciliationResult {
    Clean { checked_sequence: EventSequence },
    PendingColdProjection { hot_sequence: EventSequence, cold_sequence: EventSequence },
    DriftDetected { discrepancies: BoundedVec<ReconciliationDiscrepancy> },
    InvariantViolation { violation: RiskInvariantViolation },
    ReplayRequired { reason: ReplayRequiredReason },
    QuarantineRequired { reason: QuarantineReason },
}
```

Purpose: deterministic comparison output.

Invariants: discrepancies bounded and stably ordered.

Ownership rules: owned return value.

Serialization: cold ops/reporting.

### `RiskInvariantViolation`

```rust
pub enum RiskInvariantViolation {
    NegativeAvailable { bucket_id: CreditBucketId },
    NegativeReserved { bucket_id: CreditBucketId },
    NegativeLocked { bucket_id: CreditBucketId },
    BalanceEquationMismatch { bucket_id: CreditBucketId },
    OverConsumedReservation { reservation_id: ReservationId },
    OverReleasedReservation { reservation_id: ReservationId },
    DuplicateClientOrderReservation { client_order_id: ClientOrderId },
    RejectedOrderHasActiveReservation { reservation_id: ReservationId },
    CancelReleaseMismatch { reservation_id: ReservationId },
    ReplayStateMismatch,
    ColdLedgerDrift,
    NegativeFuturesMargin { account_id: AccountId, symbol_id: SymbolId },
    ReduceOnlyIncreasesExposure { account_id: AccountId, symbol_id: SymbolId },
}
```

Purpose: structured invariant failure.

Invariants: stable machine-readable variant.

Ownership rules: returned, logged, and possibly serialized.

Serialization: use stable codes for audit.

### `RiskEngineError`

```rust
pub enum RiskEngineError {
    Reject(RiskRejectReason),
    ReservationNotFound(ReservationId),
    ReservationOperationConflict { reservation_id: ReservationId, operation_id: OperationId },
    ArithmeticOverflow,
    InvariantViolation(RiskInvariantViolation),
    CapacityExceeded { resource: RiskCapacityResource },
    ConfigurationInvalid { code: ConfigErrorCode },
    ReplayDiverged { sequence: Option<EventSequence> },
    CacheDrift,
    ColdLedgerDrift,
}
```

Purpose: internal risk error type.

Invariants: stable codes; no secret leakage.

Ownership rules: propagated by result values, not panics.

Serialization: optional for internal audit; user API maps through reject reason.

## 40. Rust-Style Pseudocode

The following pseudocode is intentionally close to implementable Rust while avoiding production source creation.

### `reserve_for_order()`

```rust
fn reserve_for_order(&mut self, order: &Order) -> Result<RiskDecision<EventDelta>, RiskEngineError> {
    self.validate_static_order_fields(order)?;

    if let Some(existing) = self.handle_duplicate_client_order_id(order)? {
        return Ok(existing);
    }

    let decision = match (order.product_type, order.side, order.order_type) {
        (ProductType::Spot, Side::Buy, OrderType::Limit) => self.reserve_spot_buy(order),
        (ProductType::Spot, Side::Sell, OrderType::Limit) => self.reserve_spot_sell(order),
        (ProductType::Spot, Side::Buy, OrderType::Market) => self.reserve_market_buy(order),
        (ProductType::Spot, Side::Sell, OrderType::Market) => self.reserve_market_sell(order),
        (ProductType::Futures, _, _) => self.reserve_futures_margin(order),
        _ => Ok(RiskDecision::Rejected {
            reason: RiskRejectReason::UnsupportedProduct,
            event_delta: None,
        }),
    }?;

    self.validate_risk_invariants()?;
    self.metrics.record_decision_from(&decision);
    Ok(decision)
}
```

### `reserve_spot_buy()`

```rust
fn reserve_spot_buy(&mut self, order: &Order) -> Result<RiskDecision<EventDelta>, RiskEngineError> {
    let price = order.limit_price.ok_or(RiskRejectReason::InvalidPrice)?;
    let qty = order.quantity;
    ensure_positive(qty, RiskRejectReason::InvalidQuantity)?;
    ensure_positive(price, RiskRejectReason::InvalidPrice)?;

    let quote_notional = checked_mul_price_qty_round_up(price, qty)?;
    let fee_cap = self.fee_policy.max_fee_quote(order.account_id, order.symbol_id, quote_notional)?;
    let total = quote_notional.checked_add(fee_cap).ok_or(RiskEngineError::ArithmeticOverflow)?;

    let bucket_id = self.bucket_id_for(order.account_id, order.quote_asset_id);
    self.credit_buckets.reserve(bucket_id, total)?;

    let reservation_id = self.ids.reservation_id_for(order);
    let reservation = Reservation::new_active(
        reservation_id,
        order,
        bucket_id,
        order.quote_asset_id,
        total,
        fee_cap,
        self.operation_id_for_create(order),
    )?;

    self.reservations.insert_active(reservation)?;
    self.reservations.record_client_order_mapping(order.client_order_key(), reservation_id)?;

    let delta = EventDelta::reservation_created(reservation_id, total, quote_notional, fee_cap, order);
    self.risk_cache.update_after_reservation(&delta.as_reservation_delta())?;

    Ok(RiskDecision::Accepted { reservation_id, event_delta: delta })
}
```

### `reserve_spot_sell()`

```rust
fn reserve_spot_sell(&mut self, order: &Order) -> Result<RiskDecision<EventDelta>, RiskEngineError> {
    let qty = order.quantity;
    ensure_positive(qty, RiskRejectReason::InvalidQuantity)?;

    let base_fee_cap = if self.fee_policy.fee_asset_for(order) == order.base_asset_id {
        self.fee_policy.max_fee_base(order.account_id, order.symbol_id, qty)?
    } else {
        FixedAmount::ZERO
    };

    let total = qty.to_amount(order.base_asset_scale).checked_add(base_fee_cap)?;
    let bucket_id = self.bucket_id_for(order.account_id, order.base_asset_id);
    self.credit_buckets.reserve(bucket_id, total)?;

    let reservation_id = self.ids.reservation_id_for(order);
    let reservation = Reservation::new_active(reservation_id, order, bucket_id, order.base_asset_id, total, base_fee_cap, self.operation_id_for_create(order))?;
    self.reservations.insert_active(reservation)?;
    self.reservations.record_client_order_mapping(order.client_order_key(), reservation_id)?;

    let delta = EventDelta::reservation_created(reservation_id, total, qty, base_fee_cap, order);
    self.risk_cache.update_after_reservation(&delta.as_reservation_delta())?;
    Ok(RiskDecision::Accepted { reservation_id, event_delta: delta })
}
```

### `reserve_market_buy()`

```rust
fn reserve_market_buy(&mut self, order: &Order) -> Result<RiskDecision<EventDelta>, RiskEngineError> {
    if self.config.market_buy_requires_quote_budget && order.quote_budget.is_none() {
        return Ok(RiskDecision::Rejected { reason: RiskRejectReason::MissingMarketBuyBudget, event_delta: None });
    }

    let budget = order.quote_budget.ok_or(RiskRejectReason::MissingMarketBuyBudget)?;
    ensure_positive(budget, RiskRejectReason::InvalidQuantity)?;

    let fee_cap = self.fee_policy.max_fee_quote(order.account_id, order.symbol_id, budget)?;
    let total = budget.checked_add(fee_cap).ok_or(RiskEngineError::ArithmeticOverflow)?;

    let bucket_id = self.bucket_id_for(order.account_id, order.quote_asset_id);
    self.credit_buckets.reserve(bucket_id, total)?;

    let reservation_id = self.ids.reservation_id_for(order);
    let reservation = Reservation::new_active(reservation_id, order, bucket_id, order.quote_asset_id, total, fee_cap, self.operation_id_for_create(order))?;
    self.reservations.insert_active(reservation)?;
    self.reservations.record_client_order_mapping(order.client_order_key(), reservation_id)?;

    let delta = EventDelta::market_buy_budget_reserved(reservation_id, budget, fee_cap, order);
    self.risk_cache.update_after_reservation(&delta.as_reservation_delta())?;
    Ok(RiskDecision::Accepted { reservation_id, event_delta: delta })
}
```

### `reserve_market_sell()`

```rust
fn reserve_market_sell(&mut self, order: &Order) -> Result<RiskDecision<EventDelta>, RiskEngineError> {
    let qty = order.quantity;
    ensure_positive(qty, RiskRejectReason::InvalidQuantity)?;

    let fee_cap = if self.fee_policy.fee_asset_for(order) == order.base_asset_id {
        self.fee_policy.max_fee_base(order.account_id, order.symbol_id, qty)?
    } else {
        FixedAmount::ZERO
    };

    let total = qty.to_amount(order.base_asset_scale).checked_add(fee_cap)?;
    let bucket_id = self.bucket_id_for(order.account_id, order.base_asset_id);
    self.credit_buckets.reserve(bucket_id, total)?;

    let reservation_id = self.ids.reservation_id_for(order);
    let reservation = Reservation::new_active(reservation_id, order, bucket_id, order.base_asset_id, total, fee_cap, self.operation_id_for_create(order))?;
    self.reservations.insert_active(reservation)?;
    self.reservations.record_client_order_mapping(order.client_order_key(), reservation_id)?;

    let delta = EventDelta::market_sell_base_reserved(reservation_id, qty, fee_cap, order);
    self.risk_cache.update_after_reservation(&delta.as_reservation_delta())?;
    Ok(RiskDecision::Accepted { reservation_id, event_delta: delta })
}
```

### `reserve_futures_margin()`

```rust
fn reserve_futures_margin(&mut self, order: &Order) -> Result<RiskDecision<EventDelta>, RiskEngineError> {
    ensure_positive(order.quantity, RiskRejectReason::InvalidQuantity)?;

    let position = self.positions.position_view(order.account_id, order.symbol_id);
    let open_exposure = self.risk_cache.open_order_exposure(order.account_id, order.symbol_id);

    if order.reduce_only {
        self.margin.validate_reduce_only(ReduceOnlyInput { order, position, open_exposure })
            .map_err(RiskEngineError::Reject)?;
    }

    let margin_req = if order.reduce_only {
        MarginRequirement::fee_only(self.fee_policy.max_futures_fee(order)?)
    } else {
        self.margin.incremental_margin_for_order(OrderMarginInput { order, position, open_exposure })?
    };

    let total = margin_req.incremental_initial_margin.checked_add(margin_req.fee_cap)?;
    let bucket_id = self.bucket_id_for(order.account_id, margin_req.collateral_asset_id);
    self.credit_buckets.reserve(bucket_id, total)?;

    let reservation_id = self.ids.reservation_id_for(order);
    let reservation = Reservation::new_active(reservation_id, order, bucket_id, margin_req.collateral_asset_id, total, margin_req.fee_cap, self.operation_id_for_create(order))?;
    self.reservations.insert_active(reservation)?;
    self.reservations.record_client_order_mapping(order.client_order_key(), reservation_id)?;

    let delta = EventDelta::futures_margin_reserved(reservation_id, margin_req, order);
    self.risk_cache.update_after_reservation(&delta.as_reservation_delta())?;
    Ok(RiskDecision::Accepted { reservation_id, event_delta: delta })
}
```

### `consume_reservation_for_trade()`

```rust
fn consume_reservation_for_trade(&mut self, consume: ReservationConsume) -> Result<ReservationDelta, RiskEngineError> {
    let reservation = self.reservations.get_mut(consume.reservation_id)
        .ok_or(RiskEngineError::ReservationNotFound(consume.reservation_id))?;

    if reservation.has_seen_operation(consume.operation_id) {
        return reservation.previous_delta(consume.operation_id);
    }

    let delta = reservation.consume(consume)?;
    self.credit_buckets.consume_reserved(delta.bucket_id, delta.amount)?;
    self.risk_cache.update_after_reservation(&delta)?;
    self.validate_risk_invariants()?;
    Ok(delta)
}
```

### `release_reservation_for_cancel()`

```rust
fn release_reservation_for_cancel(&mut self, mut release: ReservationRelease) -> Result<ReservationDelta, RiskEngineError> {
    release.reason = ReleaseReason::Cancel;
    let reservation = self.reservations.get_mut(release.reservation_id)
        .ok_or(RiskEngineError::ReservationNotFound(release.reservation_id))?;

    if reservation.has_seen_operation(release.operation_id) {
        return reservation.previous_delta(release.operation_id);
    }

    let delta = reservation.release(release)?;
    self.credit_buckets.release_reserved(delta.bucket_id, delta.amount)?;
    self.risk_cache.update_after_reservation(&delta)?;
    self.validate_risk_invariants()?;
    Ok(delta)
}
```

### `release_reservation_for_reject()`

```rust
fn release_reservation_for_reject(&mut self, mut release: ReservationRelease) -> Result<ReservationDelta, RiskEngineError> {
    release.reason = ReleaseReason::Reject;
    let reservation = self.reservations.get_mut(release.reservation_id)
        .ok_or(RiskEngineError::ReservationNotFound(release.reservation_id))?;

    if reservation.has_seen_operation(release.operation_id) {
        return reservation.previous_delta(release.operation_id);
    }

    let delta = reservation.release(release)?;
    if !delta.after_state.is_rejected_terminal() {
        return Err(RiskEngineError::InvariantViolation(
            RiskInvariantViolation::RejectedOrderHasActiveReservation { reservation_id: delta.reservation_id }
        ));
    }
    self.credit_buckets.release_reserved(delta.bucket_id, delta.amount)?;
    self.risk_cache.update_after_reservation(&delta)?;
    self.validate_risk_invariants()?;
    Ok(delta)
}
```

### `apply_funding_risk_update()`

```rust
fn apply_funding_risk_update(&mut self, delta: FundingRiskDelta) -> Result<ReservationDelta, RiskEngineError> {
    if self.applied_funding.contains(delta.event_id) {
        return Ok(ReservationDelta::idempotent_funding(delta.event_id));
    }

    let bucket = self.credit_buckets.bucket_mut(delta.bucket_id)
        .ok_or(RiskEngineError::Reject(RiskRejectReason::InsufficientMargin))?;

    if delta.amount.is_positive() {
        bucket.apply_credit(delta.amount.abs())?;
    } else {
        bucket.apply_debit_or_lock(delta.amount.abs(), self.config.funding_debit_policy)?;
    }

    self.applied_funding.insert(delta.event_id)?;
    self.risk_cache.update_after_funding(&delta)?;
    self.validate_risk_invariants()?;
    Ok(ReservationDelta::funding_applied(delta))
}
```

### `apply_liquidation_risk_update()`

```rust
fn apply_liquidation_risk_update(&mut self, delta: LiquidationRiskDelta) -> Result<ReservationDelta, RiskEngineError> {
    if self.applied_liquidations.contains(delta.event_id) {
        return Ok(ReservationDelta::idempotent_liquidation(delta.event_id));
    }

    ensure_non_negative(delta.position_reduction_qty)?;
    ensure_non_negative(delta.fee_amount)?;

    let bucket = self.credit_buckets.bucket_mut(delta.bucket_id)
        .ok_or(RiskEngineError::InvariantViolation(RiskInvariantViolation::ColdLedgerDrift))?;

    bucket.apply_signed_collateral_delta(delta.collateral_delta)?;
    bucket.consume_or_lock_fee(delta.fee_amount)?;
    self.positions.apply_liquidation_reduction(delta.account_id, delta.symbol_id, delta.position_reduction_qty)?;

    self.applied_liquidations.insert(delta.event_id)?;
    self.risk_cache.update_after_liquidation(&delta)?;
    self.validate_risk_invariants()?;
    Ok(ReservationDelta::liquidation_applied(delta))
}
```

### `handle_duplicate_client_order_id()`

```rust
fn handle_duplicate_client_order_id(&self, order: &Order) -> Result<Option<RiskDecision<EventDelta>>, RiskEngineError> {
    let key = order.client_order_key();
    let Some(existing_id) = self.reservations.find_by_client_order(key) else {
        return Ok(None);
    };

    let existing = self.reservations.get(existing_id)
        .ok_or(RiskEngineError::InvariantViolation(
            RiskInvariantViolation::DuplicateClientOrderReservation { client_order_id: order.client_order_id }
        ))?;

    if existing.matches_order_identity(order) {
        return Ok(Some(RiskDecision::DuplicateAccepted {
            reservation_id: existing.id,
            original_event_sequence: existing.create_event_sequence.unwrap_or_default(),
        }));
    }

    Ok(Some(RiskDecision::Rejected {
        reason: RiskRejectReason::DuplicateClientOrderIdConflict,
        event_delta: None,
    }))
}
```

### `reconcile_reservations()`

```rust
fn reconcile_reservations(&self, input: ReservationReconciliationInput<'_>) -> ReconciliationResult {
    let mut discrepancies = BoundedVec::new(input.max_discrepancies);

    for bucket in input.credit_buckets.iter_stable() {
        let sum_remaining = input.reservations
            .active_for_bucket(bucket.id)
            .map(|r| r.remaining_amount())
            .checked_sum();

        if sum_remaining != bucket.reserved {
            discrepancies.push(ReconciliationDiscrepancy::ReservedMismatch {
                bucket_id: bucket.id,
                bucket_reserved: bucket.reserved,
                reservation_sum: sum_remaining,
            });
        }
    }

    for order in input.active_orders.iter_stable() {
        if input.reservations.find_by_order(order.id).is_none() {
            discrepancies.push(ReconciliationDiscrepancy::ActiveOrderMissingReservation { order_id: order.id });
        }
    }

    if discrepancies.is_empty() {
        ReconciliationResult::Clean { checked_sequence: input.sequence }
    } else {
        ReconciliationResult::DriftDetected { discrepancies }
    }
}
```

### `reconcile_with_cold_ledger()`

```rust
fn reconcile_with_cold_ledger(&self, input: ColdLedgerReconciliationInput<'_>) -> ReconciliationResult {
    if input.cold_sequence + input.allowed_lag < input.hot_sequence {
        return ReconciliationResult::PendingColdProjection {
            hot_sequence: input.hot_sequence,
            cold_sequence: input.cold_sequence,
        };
    }

    let mut discrepancies = BoundedVec::new(input.max_discrepancies);
    for hot in input.hot_buckets.iter_stable() {
        let Some(cold) = input.cold_projection.bucket(hot.id) else {
            discrepancies.push(ReconciliationDiscrepancy::MissingColdBucket { bucket_id: hot.id });
            continue;
        };
        if hot.available != cold.available || hot.reserved != cold.reserved || hot.locked != cold.locked {
            discrepancies.push(ReconciliationDiscrepancy::ColdLedgerBucketMismatch { bucket_id: hot.id });
        }
    }

    if discrepancies.is_empty() {
        ReconciliationResult::Clean { checked_sequence: input.hot_sequence.min(input.cold_sequence) }
    } else {
        ReconciliationResult::DriftDetected { discrepancies }
    }
}
```

### `detect_risk_cache_drift()`

```rust
fn detect_risk_cache_drift(&self, input: CacheDriftInput<'_>) -> ReconciliationResult {
    let mut discrepancies = BoundedVec::new(input.max_discrepancies);

    for bucket in input.credit_buckets.iter_stable() {
        let cached = input.cache.asset_state(bucket.account_id, bucket.asset_id);
        match cached {
            Some(c) if c.available == bucket.available && c.reserved == bucket.reserved && c.locked == bucket.locked => {}
            Some(_) => discrepancies.push(ReconciliationDiscrepancy::CacheBucketMismatch { bucket_id: bucket.id }),
            None => discrepancies.push(ReconciliationDiscrepancy::CacheBucketMissing { bucket_id: bucket.id }),
        }
    }

    if discrepancies.is_empty() {
        ReconciliationResult::Clean { checked_sequence: input.sequence }
    } else if input.halt_on_cache_drift {
        ReconciliationResult::QuarantineRequired { reason: QuarantineReason::CacheDrift }
    } else {
        ReconciliationResult::DriftDetected { discrepancies }
    }
}
```

### `validate_risk_invariants()`

```rust
fn validate_risk_invariants(&self) -> Result<(), RiskInvariantViolation> {
    for bucket in self.credit_buckets.iter_stable() {
        if bucket.available < FixedAmount::ZERO { return Err(RiskInvariantViolation::NegativeAvailable { bucket_id: bucket.id }); }
        if bucket.reserved < FixedAmount::ZERO { return Err(RiskInvariantViolation::NegativeReserved { bucket_id: bucket.id }); }
        if bucket.locked < FixedAmount::ZERO { return Err(RiskInvariantViolation::NegativeLocked { bucket_id: bucket.id }); }
        if bucket.total()? != bucket.available + bucket.reserved + bucket.locked + bucket.settled_adjustments {
            return Err(RiskInvariantViolation::BalanceEquationMismatch { bucket_id: bucket.id });
        }
    }

    for reservation in self.reservations.iter_stable() {
        if reservation.consumed_amount + reservation.released_amount > reservation.reserved_amount {
            return Err(RiskInvariantViolation::OverConsumedReservation { reservation_id: reservation.id });
        }
        if reservation.is_rejected_order() && !reservation.is_terminal() {
            return Err(RiskInvariantViolation::RejectedOrderHasActiveReservation { reservation_id: reservation.id });
        }
    }

    self.margin_state.validate_non_negative()?;
    self.reduce_only_state.validate_no_increase()?;
    Ok(())
}
```

### `replay_risk_state()`

```rust
fn replay_risk_state(events: impl Iterator<Item = EngineEventView>, config: RiskEngineConfig) -> Result<RiskState, RiskEngineError> {
    let mut state = RiskState::empty(config);
    let mut expected_next = EventSequence::first();

    for event in events {
        if event.sequence != expected_next {
            return Err(RiskEngineError::ReplayDiverged { sequence: Some(event.sequence) });
        }
        if !event.hash_chain_valid_if_present() {
            return Err(RiskEngineError::ReplayDiverged { sequence: Some(event.sequence) });
        }

        match event.kind {
            EngineEventKind::ReservationCreated(payload) => state.apply_reservation_created(payload)?,
            EngineEventKind::ReservationConsumed(payload) => state.consume_reservation_for_trade(payload.into())?,
            EngineEventKind::ReservationReleased(payload) => state.release_reservation(payload.into())?,
            EngineEventKind::FundingApplied(payload) => state.apply_funding_risk_update(payload.into())?,
            EngineEventKind::LiquidationApplied(payload) => state.apply_liquidation_risk_update(payload.into())?,
            EngineEventKind::LedgerAdjustment(payload) => state.apply_ledger_adjustment(payload.into())?,
            _ => state.apply_non_risk_event_if_relevant(event)?,
        }

        if state.config.replay_validate_every_event {
            state.validate_risk_invariants()?;
        }
        expected_next = expected_next.next();
    }

    state.validate_risk_invariants()?;
    state.rebuild_risk_cache()?;
    Ok(state)
}
```

## 41. Testing Strategy

Testing must prove accounting safety, idempotency, determinism, and replay equivalence.

Test types:

- unit tests for pure functions and state machines;
- integration tests for order/reservation/trade/cancel/reject flows;
- property tests for randomized command sequences;
- replay tests for crash and duplicate scenarios;
- determinism tests for same input/same output;
- performance benchmarks for latency and throughput.

All tests must use fixed-point integer values and deterministic seeds. Golden-vector tests should compare stable serialized event payloads where event schemas exist.

## 42. Unit Tests

| Test ID | Name | Purpose | Required Assertions |
| --- | --- | --- | --- |
| RISK-UNIT-001 | spot buy reservation | Quote plus fee reserved. | available decreases; reserved increases; event delta exact. |
| RISK-UNIT-002 | spot sell reservation | Base reserved. | base available decreases; no quote mutation. |
| RISK-UNIT-003 | fee reservation | Conservative fee cap reserved. | fee cap rounded up; unused fee releasable. |
| RISK-UNIT-004 | market buy quote budget | Budget required and reserved. | missing budget rejects; budget plus fee held. |
| RISK-UNIT-005 | futures margin | Initial margin and fee reserved. | initial >= maintenance; margin nonnegative. |
| RISK-UNIT-006 | duplicate reservation prevention | Same client order does not reserve twice. | second call returns existing ID; balances unchanged. |
| RISK-UNIT-007 | release once | Release idempotent. | retry does not increase available twice. |
| RISK-UNIT-008 | consume once | Consume idempotent. | retry does not reduce reserved twice. |
| RISK-UNIT-009 | invariant violation detection | Invalid state is detected. | each invariant variant has negative test. |
| RISK-UNIT-010 | reduce-only rejection | Reduce-only cannot increase exposure. | wrong-side or oversize reduce-only rejects. |
| RISK-UNIT-011 | funding duplicate | Funding event applied once. | duplicate no-op. |
| RISK-UNIT-012 | liquidation duplicate | Liquidation event applied once. | duplicate no-op. |

## 43. Integration Tests

| Test ID | Scenario | Flow | Required Assertions |
| --- | --- | --- | --- |
| RISK-INT-001 | order accepted and reserved | order -> reserve -> event delta | Book Core admission has reservation ID. |
| RISK-INT-002 | trade consumes reservation | reserve -> trade -> consume | reserved decreases by fill amount. |
| RISK-INT-003 | cancel releases reservation | reserve -> cancel -> release | remaining returns to available once. |
| RISK-INT-004 | reject leaves no active reservation | staged reserve -> reject release | no active hold remains. |
| RISK-INT-005 | partial fill consume/release | reserve -> partial fill -> cancel | consumed + released equals reserved. |
| RISK-INT-006 | duplicate client_order_id retry | reserve -> retry same request | no double reserve. |
| RISK-INT-007 | duplicate conflict | reserve -> retry changed request | stable duplicate conflict reject. |
| RISK-INT-008 | funding risk update | reserve -> funding debit/credit | bucket and cache update deterministically. |
| RISK-INT-009 | liquidation risk update | position -> liquidation delta | exposure reduced; margin valid. |
| RISK-INT-010 | cold ledger reconciliation | event projection -> ledger projection compare | clean or lagged result as expected. |

## 44. Property Tests

| Test ID | Property | Generator | Required Assertions |
| --- | --- | --- | --- |
| RISK-PROP-001 | no negative available balance | random reserves/consumes/releases | available never below zero. |
| RISK-PROP-002 | balance conservation | random lifecycle sequences | total equation holds. |
| RISK-PROP-003 | no double reserve | duplicate client IDs | bucket reserved changes only once. |
| RISK-PROP-004 | no double release | repeated release operation IDs | available increases once. |
| RISK-PROP-005 | no double consume | repeated consume operation IDs | reserved decreases once. |
| RISK-PROP-006 | replay equivalence | generated event streams | live state equals replay state. |
| RISK-PROP-007 | reduce-only safety | positions and reduce orders | exposure never increases. |
| RISK-PROP-008 | funding/liquidation idempotency | duplicate deltas | applied sets prevent double mutation. |

## 45. Replay Tests

| Test ID | Scenario | Required Assertions |
| --- | --- | --- |
| RISK-REPLAY-001 | replay reservation lifecycle | created/consumed/released reconstruct terminal reservation. |
| RISK-REPLAY-002 | crash after reserve before response | retry returns existing reservation from event state. |
| RISK-REPLAY-003 | crash after consume before response | retry consume is idempotent. |
| RISK-REPLAY-004 | duplicate retry after unknown gateway response | no double reserve and same result returned. |
| RISK-REPLAY-005 | cold ledger reconciliation after replay | replayed hot state matches ledger projection at watermark. |
| RISK-REPLAY-006 | funding replay | funding deltas applied once. |
| RISK-REPLAY-007 | liquidation replay | liquidation deltas applied once. |
| RISK-REPLAY-008 | sequence gap | replay rejects missing sequence. |
| RISK-REPLAY-009 | hash mismatch | replay rejects invalid hash chain. |

## 46. Determinism Tests

| Test ID | Scenario | Required Assertions |
| --- | --- | --- |
| RISK-DET-001 | same order same state | two engines, same state | identical `RiskDecision`. |
| RISK-DET-002 | replay vs live | live command sequence then replay events | identical buckets/reservations/cache. |
| RISK-DET-003 | no clock dependency | fixed event inputs with changed wall clock | identical output. |
| RISK-DET-004 | stable rounding | boundary fixed-point values | exact expected integer result. |
| RISK-DET-005 | stable reconciliation ordering | same discrepancy set shuffled | result discrepancies sorted identically. |

## 47. Performance Benchmarks

| Benchmark ID | Name | Measurement | Gate |
| --- | --- | --- | --- |
| RISK-BENCH-001 | reserve_for_order latency | p50/p95/p99/p999 ns | threshold set by CI profile. |
| RISK-BENCH-002 | reservation consume latency | p50/p95/p99/p999 ns | no regression over baseline. |
| RISK-BENCH-003 | release latency | p50/p95/p99/p999 ns | no regression over baseline. |
| RISK-BENCH-004 | risk cache lookup latency | p50/p95/p99 ns | O(1) expected. |
| RISK-BENCH-005 | reconciliation batch throughput | reservations/sec | meets operations target. |
| RISK-BENCH-006 | replay throughput | events/sec | supports recovery objective. |
| RISK-BENCH-007 | allocation count | allocations/op | zero or approved bounded count on hot path. |
| RISK-BENCH-008 | capacity pressure | near-full bounded maps | deterministic error and no unbounded allocation. |

Benchmark rules:

- use release builds;
- pin deterministic input data;
- record CPU and compiler version in CI artifact;
- fail on accidental float use, allocation regression, or forbidden dependency introduction where tooling supports it.

## 48. Dependency Policy

Allowed in hot risk modules:

- Rust standard/core collections only if bounded by wrapper or preallocation policy;
- workspace fixed-point, ID, domain, and event crates;
- small error derive macros;
- feature-gated serialization for cold paths and tests.

Forbidden in hot risk modules:

- database clients;
- Kafka or message broker clients;
- Redis or external caches;
- HTTP/gRPC clients;
- cloud SDKs;
- async runtimes;
- floating-point math crates;
- wall-clock time dependencies;
- random generators;
- logging/metrics exporters that block or allocate unbounded memory.

Any dependency addition must answer:

- Is it deterministic?
- Can it allocate unbounded memory?
- Can it perform IO?
- Does it use floats for financial values?
- Is it needed in hot path or only in cold/test feature?
- Can replay run without it?

## 49. Implementation Sequence

1. Create `crates/hermes-risk` skeleton and `Cargo.toml` with only allowed dependencies.
2. Define fixed public types in `config.rs`, `error.rs`, `reservation.rs`, `credit_bucket.rs`, `risk_cache.rs`, and `margin.rs`.
3. Add trait contracts in `lib.rs` and module files.
4. Implement pure fixed-point fee and notional helper functions.
5. Implement `CreditBucket` reserve/consume/release with invariant checks.
6. Implement `Reservation` lifecycle with operation ID idempotency.
7. Implement duplicate client order map and conflict checks.
8. Implement spot buy/sell and market buy/sell reservation orchestration.
9. Implement futures margin and reduce-only risk handling.
10. Implement funding and liquidation delta application.
11. Implement risk cache updates and cache drift detection.
12. Implement reconciliation comparisons.
13. Implement replay state builder.
14. Add unit tests for every state transition.
15. Add integration tests for accepted/trade/cancel/reject/partial fill flows.
16. Add property tests for no negative balances, conservation, idempotency, and replay equivalence.
17. Add replay crash-point tests.
18. Add benchmark harness.
19. Add dependency audit CI rule.
20. Review against HES invariants and module contract.

## 50. Codex Implementation Tasks

Task set for a future implementation agent:

- `RISK-CODEX-001`: Create crate skeleton only, with empty modules and public re-exports.
- `RISK-CODEX-002`: Implement `RiskEngineConfig` and validation tests.
- `RISK-CODEX-003`: Implement `RiskRejectReason`, `RiskEngineError`, and `RiskInvariantViolation` with stable codes.
- `RISK-CODEX-004`: Implement `CreditBucket` arithmetic and invariants.
- `RISK-CODEX-005`: Implement `Reservation` lifecycle and idempotent consume/release.
- `RISK-CODEX-006`: Implement duplicate `client_order_id` lookup contract.
- `RISK-CODEX-007`: Implement spot buy/sell reservation paths.
- `RISK-CODEX-008`: Implement market buy/sell reservation paths.
- `RISK-CODEX-009`: Implement fee cap calculation interfaces using fixed-point only.
- `RISK-CODEX-010`: Implement futures margin and reduce-only checks.
- `RISK-CODEX-011`: Implement funding risk updater.
- `RISK-CODEX-012`: Implement liquidation risk updater.
- `RISK-CODEX-013`: Implement risk cache and drift detection.
- `RISK-CODEX-014`: Implement reconciliation result generation.
- `RISK-CODEX-015`: Implement replay builder from event views.
- `RISK-CODEX-016`: Add unit test matrix.
- `RISK-CODEX-017`: Add integration and replay tests.
- `RISK-CODEX-018`: Add property tests.
- `RISK-CODEX-019`: Add benchmarks and allocation checks.
- `RISK-CODEX-020`: Run dependency audit and review checklist.

Task constraints:

- do not create production ledger or Book Core code in `hermes-risk`;
- do not introduce floats;
- do not add IO dependencies;
- do not skip replay tests;
- do not make idempotency best-effort.

## 51. Review Checklist

Architecture:

- [ ] Risk decision is local to Book Core boundary.
- [ ] No DB, Kafka, Redis, HTTP, gRPC, cloud SDK, or filesystem dependency exists in hot path.
- [ ] Fixed-point integer types are used for all financial calculations.
- [ ] No float conversion exists in risk/margin/reservation code.
- [ ] Hot and cold risk responsibilities are separate.

Reservation safety:

- [ ] No double reserve.
- [ ] No double spend.
- [ ] No negative available balance.
- [ ] Consume is idempotent.
- [ ] Release is idempotent.
- [ ] Duplicate `client_order_id` cannot create a second reservation.
- [ ] Reject leaves no active reservation.
- [ ] Cancel releases remaining amount exactly once.

Replay and audit:

- [ ] Every financial mutation produces replayable event data.
- [ ] Replay reconstructs identical reservations and buckets.
- [ ] Cold ledger reconciliation compares compatible sequence watermarks.
- [ ] Funding and liquidation deltas are event-backed and idempotent.
- [ ] Error codes and reject reasons are stable.

Testing:

- [ ] Unit test matrix implemented.
- [ ] Integration test matrix implemented.
- [ ] Property tests implemented.
- [ ] Replay crash-point tests implemented.
- [ ] Determinism tests implemented.
- [ ] Performance benchmarks implemented.

Operations:

- [ ] Metrics labels are bounded.
- [ ] Logs avoid sensitive data.
- [ ] Invariant violations fail closed.
- [ ] Drift detection has clear quarantine/replay workflow.
- [ ] Dependency audit passes.

## 52. Completion Criteria

`hermes-risk` implementation is complete only when:

- all required modules exist with documented responsibilities;
- all public traits are implemented or intentionally mocked in tests;
- spot buy/sell and market buy/sell reservation paths pass tests;
- fee reservation is deterministic and conservative;
- futures margin and reduce-only checks pass tests;
- funding and liquidation delta application is idempotent and replayable;
- duplicate `client_order_id` cannot double reserve;
- consume/release are idempotent;
- invariant table checks are implemented;
- replay reconstructs identical reservation, bucket, idempotency, and cache state;
- cold ledger reconciliation detects clean, lagged, and drift states;
- risk cache drift detection can quarantine or rebuild according to config;
- all unit, integration, property, replay, determinism, and benchmark gates pass;
- dependency audit proves no forbidden hot-path dependency;
- review checklist is complete;
- no HES behavior has been expanded or contradicted.

## Explicit Invariant Table

| Invariant ID | Invariant | Enforcement Point | Failure Response |
| --- | --- | --- | --- |
| RISK-INV-001 | `available >= 0` | credit bucket mutation and invariant validation | reject before mutation or halt on corruption. |
| RISK-INV-002 | `reserved >= 0` | reserve/consume/release | halt on underflow. |
| RISK-INV-003 | `locked >= 0` | lock/unlock/liquidation/funding | halt on underflow. |
| RISK-INV-004 | `total = available + reserved + locked + settled_adjustments` | bucket validation and reconciliation | drift/invariant violation. |
| RISK-INV-005 | reservation cannot be consumed more than once for same operation | reservation lifecycle | idempotent same result. |
| RISK-INV-006 | reservation cannot be released more than once for same operation | reservation lifecycle | idempotent same result. |
| RISK-INV-007 | `consumed + released <= reserved_amount` | lifecycle and replay | reject mutation or halt. |
| RISK-INV-008 | duplicate `client_order_id` cannot create second reservation | engine duplicate handler | return existing or conflict reject. |
| RISK-INV-009 | rejected order cannot leave active reservation | reject path and reconciliation | reject release required or halt. |
| RISK-INV-010 | cancelled order releases remaining reservation exactly once | cancel release | idempotent terminal release. |
| RISK-INV-011 | replay reconstructs same available/reserved state | replay tests and recovery | replay divergence. |
| RISK-INV-012 | cold ledger projection eventually reconciles with hot risk state | reconciliation | pending, drift, quarantine. |
| RISK-INV-013 | futures margin cannot be negative | margin and liquidation modules | reject/halt. |
| RISK-INV-014 | reduce-only order must not increase exposure | margin reduce-only validation | reject. |
| RISK-INV-015 | all financial mutations are auditable | event delta creation | no external success without event data. |
| RISK-INV-016 | no double spend across buckets | bucket reserve and consume | insufficient available reject. |
| RISK-INV-017 | fee cap cannot under-reserve | fee calculation | conservative round-up. |
| RISK-INV-018 | funding delta applied once | funding updater applied set | idempotent duplicate. |
| RISK-INV-019 | liquidation delta applied once | liquidation updater applied set | idempotent duplicate. |
| RISK-INV-020 | cache cannot override authoritative state | drift detection | rebuild/quarantine. |
