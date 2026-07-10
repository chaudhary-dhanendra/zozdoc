# Book Core Implementation

## 1. Purpose

Define implementation-grade Rust guidance for book core implementation while preserving HES as the source of truth. This file specifies crate responsibilities, module boundaries, contracts, deterministic control flow, pseudocode, tests, benchmarks, and review gates.

## 2. Related HES volumes

- HES volumes for matching, risk, clearing, settlement, events, replay, operations, observability, and security.
- HES invariants: fixed-point integers only; no floats in matching/risk/settlement; bounded queues; immutable hash-chained EngineEvents; event append before externally visible success; deterministic replay; no DB/Kafka in the hot path; all financial mutations auditable.

## 3. Implementation scope

Covers crate structure, single-writer runtime, sequence assignment, inbound ring consumption, validation boundary, risk/matching/clearing calls, EngineEvent construction, append-before-success, response publication, snapshots, replay rebuild, memory ownership, cache-line layout, arenas, pools, node/level lifecycle, event buffers, risk/settlement views, shutdown, crash recovery, deterministic replay, counters, implementation sequence, forbidden dependencies, and unsafe-code policy. Unsafe is forbidden initially; any later unsafe requires isolated module, proof comment, Miri tests, benchmark justification, and safe fallback.

## 4. Non-goals

- No production Rust source files are created by this handbook.
- No expansion of HES behavior or changes to HES obligations.
- No DOCX/PDF generation.
- No unbounded caches, hidden threads, wall-clock ordering, random IDs, floats, SQL, or Kafka in hot-path modules.

## 5. Target crates

- `hermes-book`
- `hermes-matching`
- `hermes-risk`
- `hermes-clearing`
- `hermes-events`
- `hermes-replay`
- `hermes-fixed`
- `hermes-ids`

## 6. Module layout

Use `lib.rs` for public contracts and re-exports. Use crate-local `config.rs`, `error.rs`, `types.rs`, `metrics.rs`, `tests/`, and domain modules. Hot modules receive dependencies as traits and own their mutable state. Cold modules perform IO, config loading, process supervision, and external protocol adaptation.

## 7. File layout

```text
crates/
  hermes-book/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-matching/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-risk/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-clearing/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-events/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-replay/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-fixed/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-ids/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
```

## 8. Key traits

- `BookCoreRuntime`: owns the single-writer loop for one book partition.
- `OrderProcessor`: validates and processes commands inside the writer.
- `MatchingEngine`: mutates `BookState` and returns deterministic fills/cancels.
- `BookStateStore`: in-memory order/price-level access with arena/pool ownership.
- `EventAppender`: synchronously appends hash-chained EngineEvents before success.
- `SnapshotProvider`: creates/restores deterministic snapshots.
- `Replayable`: applies events during recovery without external IO.
- `ResponsePublisher`: sends acks/rejects after durable append.

## 9. Key structs/enums

`BookCore`, `BookState`, `BookCoreConfig`, `BookCoreCommand`, `BookCoreResponse`, `BookCoreMetrics`, `ProcessOrderContext`, `BookCoreError`, `OrderNode`, `PriceLevel`, `OrderArena`, `LevelPool`, `EventBuffer`, `RiskCacheView`, `SettlementDeltaView`, `BookSequence`, `SnapshotCursor`. Use cache-line padding for counters and ring indices; store hot fields contiguously; keep cold metadata out of matching nodes.

## 10. Error types

Use domain-specific error enums with stable reject/recovery codes, e.g. `InvalidCommand`, `ArithmeticOverflow`, `InvariantViolation`, `Duplicate`, `InsufficientAvailable`, `QueueFull`, `EventAppendFailed`, `HashChainMismatch`, `SnapshotInvalid`, `ReplayDiverged`, `ExternalDependencyUnavailable`, and `ConfigurationInvalid`. Errors must include machine-actionable codes and avoid secret/account-sensitive debug leakage.

## 11. Control flow

The Book Core is a single mutable owner per symbol/book. It consumes inbound ring commands, assigns book-local sequence, performs boundary validation, calls risk reservation, matching, and clearing in a deterministic order, builds EngineEvents, appends them, publishes responses, updates counters, and periodically snapshots. No shared mutable state crosses the loop; external components communicate by bounded queues and immutable events. Order nodes are allocated from arenas before steady state, recycled through object pools, and never heap-allocated in matching. Price levels are inserted/removed deterministically. Shutdown drains or rejects according to policy; crash recovery replays from verified snapshots/events and compares deterministic hashes. Forbidden dependencies: SQL drivers, Kafka clients, HTTP servers, async executors in matching loop, JSON serializers, system time, random generators, floats, global mutable state, and logging calls in the inner match loop.

## 12. Rust-style pseudocode

```rust
fn run_book_core_loop(core) { pin_thread(core.cpu); while core.running { match inbound.try_pop_batch() { cmds => for cmd in cmds { process_command(core, cmd); }, empty => core.idle_strategy.poll() } maybe_snapshot(core); } shutdown_drain_or_reject(core); }
fn process_command(core, cmd) { let seq=assign_book_sequence(core); match cmd { Place(o)=>process_order(core, seq, o), Cancel(c)=>process_cancel(core, seq, c), Admin(a)=>process_admin(core, seq, a) } }
fn process_order(core, seq, order) { let ctx=ProcessOrderContext::new(seq, order); call_risk(core.risk_view, &ctx)?; let fills=call_matching(&mut core.book_state, &ctx)?; let deltas=call_clearing(core.clearing, &ctx, &fills)?; let ev=append_engine_event(core.events, seq, ctx, fills, deltas)?; publish_response(core.responses, ev); }
fn assign_book_sequence(core) -> BookSequence { core.next_seq += 1; core.next_seq }
fn call_risk(risk, ctx) { risk.reserve_or_reject(ctx) }
fn call_matching(state, ctx) { matching.match_order(state, ctx) }
fn call_clearing(clearing, ctx, fills) { clearing.build_deltas(ctx, fills) }
fn append_engine_event(log, seq, ctx, fills, deltas) { let ev=EngineEvent::from_parts(seq, ctx, fills, deltas); log.append_and_fsync_policy(ev) }
fn publish_response(pub, ev) { pub.try_publish(BookCoreResponse::from_event(ev)); }
fn maybe_snapshot(core) { if core.metrics.events_since_snapshot >= core.cfg.snapshot_every { core.snapshot.write(core.book_state, core.last_event_hash); } }
fn replay_book_core(snapshot, events) { let mut core=restore(snapshot); for ev in verify_hash_chain(events) { core.apply_replay_event(ev); } assert_eq!(core.state_hash(), expected); }
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
