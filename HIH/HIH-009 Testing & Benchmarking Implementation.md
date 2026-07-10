# Testing & Benchmarking Implementation

## 1. Purpose

Define implementation-grade Rust guidance for testing & benchmarking implementation while preserving HES as the source of truth. This file specifies crate responsibilities, module boundaries, contracts, deterministic control flow, pseudocode, tests, benchmarks, and review gates.

## 2. Related HES volumes

- HES volumes for matching, risk, clearing, settlement, events, replay, operations, observability, and security.
- HES invariants: fixed-point integers only; no floats in matching/risk/settlement; bounded queues; immutable hash-chained EngineEvents; event append before externally visible success; deterministic replay; no DB/Kafka in the hot path; all financial mutations auditable.

## 3. Implementation scope

Covers unit, integration, property, replay, fuzz, chaos, golden vector, performance, latency, memory, deterministic certification, event hash validation, gateway load, Book Core/risk/clearing/wallet/futures benchmarks, and CI execution.

## 4. Non-goals

- No production Rust source files are created by this handbook.
- No expansion of HES behavior or changes to HES obligations.
- No DOCX/PDF generation.
- No unbounded caches, hidden threads, wall-clock ordering, random IDs, floats, SQL, or Kafka in hot-path modules.

## 5. Target crates

- `hermes-tests`
- `hermes-benches`
- `hermes-fixtures`
- `hermes-golden-vectors`

## 6. Module layout

Use `lib.rs` for public contracts and re-exports. Use crate-local `config.rs`, `error.rs`, `types.rs`, `metrics.rs`, `tests/`, and domain modules. Hot modules receive dependencies as traits and own their mutable state. Cold modules perform IO, config loading, process supervision, and external protocol adaptation.

## 7. File layout

```text
crates/
  hermes-tests/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-benches/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-fixtures/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-golden-vectors/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
```

## 8. Key traits

Define harness traits such as `GoldenVectorRunner`, `ReplayHarness`, `FuzzTarget`, `ChaosScenario`, `BenchmarkCase`, `LatencyRecorder`, and `FixtureFactory`.

## 9. Key structs/enums

`TestEngine`, `CommandTrace`, `ExpectedEventTrace`, `ReplayResult`, `FuzzCorpus`, `ChaosPlan`, `BenchmarkReport`, `AllocationReport`, `CertificationReport`.

## 10. Error types

Use domain-specific error enums with stable reject/recovery codes, e.g. `InvalidCommand`, `ArithmeticOverflow`, `InvariantViolation`, `Duplicate`, `InsufficientAvailable`, `QueueFull`, `EventAppendFailed`, `HashChainMismatch`, `SnapshotInvalid`, `ReplayDiverged`, `ExternalDependencyUnavailable`, and `ConfigurationInvalid`. Errors must include machine-actionable codes and avoid secret/account-sensitive debug leakage.

## 11. Control flow

Tests are layered: unit tests inside crates, integration harness across gateway/book/risk/clearing, property tests for command sequences, replay tests for event logs, fuzzers for parsers/serializers, chaos tests for crash/failover, golden vectors for fixed outputs, and benchmarks for latency/memory. CI runs fast checks per PR and scheduled exhaustive fuzz/chaos/perf suites.

## 12. Rust-style pseudocode

```rust
fn run_golden_vector(v){ result=harness.run(v.input); assert_eq!(result.events, v.expected_events); assert_eq!(result.hash, v.expected_hash); }
fn assert_replay_equivalence(trace){ live=run_live(trace); replay=replay_events(live.events); assert_eq!(live.state_hash, replay.state_hash); }
fn fuzz_order_parser(bytes){ if let Ok(o)=parse(bytes){ assert_valid_or_rejected(o); } }
fn benchmark_book_core(){ feed_prebuilt_commands_no_alloc(); record_latency_throughput_allocs(); }
fn benchmark_matching(){ run_crossing_and_non_crossing_cases(); }
fn benchmark_replay(){ load_segments(); measure_events_per_second(); }
fn run_chaos_failover_test(){ kill_after_append_points(); restart_from_snapshot_events(); assert_replay_equivalence(trace); }
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
