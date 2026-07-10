# Futures & Liquidation Implementation

## 1. Purpose

Define implementation-grade Rust guidance for futures & liquidation implementation while preserving HES as the source of truth. This file specifies crate responsibilities, module boundaries, contracts, deterministic control flow, pseudocode, tests, benchmarks, and review gates.

## 2. Related HES volumes

- HES volumes for matching, risk, clearing, settlement, events, replay, operations, observability, and security.
- HES invariants: fixed-point integers only; no floats in matching/risk/settlement; bounded queues; immutable hash-chained EngineEvents; event append before externally visible success; deterministic replay; no DB/Kafka in the hot path; all financial mutations auditable.

## 3. Implementation scope

Covers positions, margin, funding, mark/index prices, liquidation, ADL, portfolio/cross/isolated modes, monitoring, stress hooks, replay, and certification.

## 4. Non-goals

- No production Rust source files are created by this handbook.
- No expansion of HES behavior or changes to HES obligations.
- No DOCX/PDF generation.
- No unbounded caches, hidden threads, wall-clock ordering, random IDs, floats, SQL, or Kafka in hot-path modules.

## 5. Target crates

- `hermes-futures`
- `hermes-margin`
- `hermes-liquidation`
- `hermes-risk`
- `hermes-oracle`
- `hermes-events`

## 6. Module layout

Use `lib.rs` for public contracts and re-exports. Use crate-local `config.rs`, `error.rs`, `types.rs`, `metrics.rs`, `tests/`, and domain modules. Hot modules receive dependencies as traits and own their mutable state. Cold modules perform IO, config loading, process supervision, and external protocol adaptation.

## 7. File layout

```text
crates/
  hermes-futures/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-margin/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-liquidation/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-risk/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-oracle/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-events/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
```

## 8. Key traits

- `FuturesEngine`; `PositionService`; `MarginEngine`; `FundingEngine`; `MarkPriceService`; `LiquidationEngine`; `AdlEngine`; `PortfolioMarginEngine`.

## 9. Key structs/enums

`Position`, `PositionDelta`, `MarginMode`, `MarginRequirement`, `FundingRate`, `MarkPrice`, `IndexPrice`, `LiquidationCandidate`, `LiquidationOrder`, `AdlRank`, `StressScenario`, `CertificationVector`.

## 10. Error types

Use domain-specific error enums with stable reject/recovery codes, e.g. `InvalidCommand`, `ArithmeticOverflow`, `InvariantViolation`, `Duplicate`, `InsufficientAvailable`, `QueueFull`, `EventAppendFailed`, `HashChainMismatch`, `SnapshotInvalid`, `ReplayDiverged`, `ExternalDependencyUnavailable`, and `ConfigurationInvalid`. Errors must include machine-actionable codes and avoid secret/account-sensitive debug leakage.

## 11. Control flow

Trade events update positions using fixed-point quantity/notional math. Margin engine supports cross, isolated, and portfolio margin via deterministic risk arrays. Mark/index services ingest oracle data at cold boundaries and publish versioned fixed-point prices. Funding events debit/credit positions. Risk monitor evaluates maintenance margin; liquidation executes partial reductions before full liquidation, posts insurance fund deltas, and triggers ADL ranking only when configured buffers fail. All derivatives state rebuilds from events and certification vectors.

## 12. Rust-style pseudocode

```rust
fn open_position(order){ reserve_futures_margin(order); submit_to_book(order); }
fn update_position_after_trade(pos, fill){ pos.qty += signed(fill.qty); pos.entry = weighted_fixed(pos.entry, fill.price); realize_pnl_if_reducing(pos, fill); }
fn calculate_margin(portfolio){ return margin_engine.requirement(portfolio, mark_prices); }
fn apply_funding(rate){ for pos in open_positions { post_funding_delta(pos, rate); } }
fn evaluate_liquidation(acct){ if equity(acct) < maintenance(acct) { enqueue_candidate(acct); } }
fn execute_partial_liquidation(c){ size=deterministic_step(c); submit_liquidation_order(size); }
fn rank_adl_candidates(){ sort_by_pnl_leverage_account_id(); }
fn execute_adl(candidate){ reduce_against_ranked_counterparties(candidate); emit_adl_events(); }
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
