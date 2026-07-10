# Deployment & Operations Implementation

## 1. Purpose

Define implementation-grade Rust guidance for deployment & operations implementation while preserving HES as the source of truth. This file specifies crate responsibilities, module boundaries, contracts, deterministic control flow, pseudocode, tests, benchmarks, and review gates.

## 2. Related HES volumes

- HES volumes for matching, risk, clearing, settlement, events, replay, operations, observability, and security.
- HES invariants: fixed-point integers only; no floats in matching/risk/settlement; bounded queues; immutable hash-chained EngineEvents; event append before externally visible success; deterministic replay; no DB/Kafka in the hot path; all financial mutations auditable.

## 3. Implementation scope

Covers environment layout, configuration/secrets loading, local/staging/production, Docker, Kubernetes, CPU pinning, node affinity, observability, logs/metrics/traces, health/readiness/liveness, failover, backup/restore, release/rollback, and runbooks.

## 4. Non-goals

- No production Rust source files are created by this handbook.
- No expansion of HES behavior or changes to HES obligations.
- No DOCX/PDF generation.
- No unbounded caches, hidden threads, wall-clock ordering, random IDs, floats, SQL, or Kafka in hot-path modules.

## 5. Target crates

- `hermes-config`
- `hermes-observability`
- `hermes-admin`
- `deployment manifests`
- `scripts`

## 6. Module layout

Use `lib.rs` for public contracts and re-exports. Use crate-local `config.rs`, `error.rs`, `types.rs`, `metrics.rs`, `tests/`, and domain modules. Hot modules receive dependencies as traits and own their mutable state. Cold modules perform IO, config loading, process supervision, and external protocol adaptation.

## 7. File layout

```text
crates/
  hermes-config/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-observability/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-admin/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  deployment manifests/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  scripts/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
```

## 8. Key traits

- `ConfigLoader`; `SecretLoader`; `HealthService`; `ReadinessGate`; `BackupService`; `RestoreService`; `ReleaseController`; `RunbookExecutor`; `MetricsExporter`; `TraceInitializer`.

## 9. Key structs/enums

`EnvironmentConfig`, `SecretRef`, `HealthStatus`, `ReadinessStatus`, `BackupManifest`, `RestorePlan`, `ReleaseVersion`, `RollbackPlan`, `NodePlacement`, `CpuPinningPolicy`.

## 10. Error types

Use domain-specific error enums with stable reject/recovery codes, e.g. `InvalidCommand`, `ArithmeticOverflow`, `InvariantViolation`, `Duplicate`, `InsufficientAvailable`, `QueueFull`, `EventAppendFailed`, `HashChainMismatch`, `SnapshotInvalid`, `ReplayDiverged`, `ExternalDependencyUnavailable`, and `ConfigurationInvalid`. Errors must include machine-actionable codes and avoid secret/account-sensitive debug leakage.

## 11. Control flow

Local, staging, and production differ only by config/secrets and capacity. Startup loads validated config, then secrets, then opens event logs/snapshots, then health/readiness. Docker images are minimal and reproducible. Kubernetes pins Book Core pods to dedicated CPUs/nodes, uses node affinity and persistent volumes, and exposes health/readiness/liveness endpoints. Observability ships logs/metrics/traces. Backup copies snapshots plus verified event segments; restore validates hash chain. Releases are canary/blue-green with rollback checklists and runbooks.

## 12. Rust-style pseudocode

```rust
# config shape
environment: production
books: [{symbol: BTC-USD, cpu: 2, inbound_queue: 65536, snapshot_every: 1000000}]
event_log: {path: /var/lib/hermes/events, fsync: every_event}
secrets: {provider: vault, api_key_path: hermes/prod/api}
observability: {metrics: 0.0.0.0:9100, tracing: otlp}

# Kubernetes skeleton
apiVersion: apps/v1
kind: Deployment
spec: {template: {spec: {nodeSelector: {hermes.net/hotpath: "true"}, containers: [{name: book-core, readinessProbe: {httpGet: {path: /ready, port: 8080}}}]}}}

fn readiness(){ ready = event_log_open && replay_complete && router_bound && hash_head_verified; }
fn health(){ return process_alive && metrics_exporter_alive; }
fn rollback_checklist(){ stop_new_admission(); verify_old_binary_compatible(); restore_config(); replay_to_hash(); reopen_gateway(); }
```

## 13. Configuration

Environment variables may select environment and secret references only; they must not carry long-lived secrets. Config files are schema-validated, versioned, and immutable per release. Include CPU pinning, node affinity, queue sizes, fsync policy, TLS, auth providers, retention, backup cadence, and observability endpoints.

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
