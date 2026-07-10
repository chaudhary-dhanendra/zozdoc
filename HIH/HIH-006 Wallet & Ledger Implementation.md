# Wallet & Ledger Implementation

## 1. Purpose

Define implementation-grade Rust guidance for wallet & ledger implementation while preserving HES as the source of truth. This file specifies crate responsibilities, module boundaries, contracts, deterministic control flow, pseudocode, tests, benchmarks, and review gates.

## 2. Related HES volumes

- HES volumes for matching, risk, clearing, settlement, events, replay, operations, observability, and security.
- HES invariants: fixed-point integers only; no floats in matching/risk/settlement; bounded queues; immutable hash-chained EngineEvents; event append before externally visible success; deterministic replay; no DB/Kafka in the hot path; all financial mutations auditable.

## 3. Implementation scope

Covers wallet service, ledger projection, double-entry builder, deposits, withdrawals, transfers, treasury, HSM/MPC boundary, maker-checker, reconciliation, statements, insurance fund, fee/revenue accounting, replay, and audit evidence.

## 4. Non-goals

- No production Rust source files are created by this handbook.
- No expansion of HES behavior or changes to HES obligations.
- No DOCX/PDF generation.
- No unbounded caches, hidden threads, wall-clock ordering, random IDs, floats, SQL, or Kafka in hot-path modules.

## 5. Target crates

- `hermes-wallet`
- `hermes-ledger`
- `hermes-treasury`
- `hermes-compliance`
- `hermes-events`

## 6. Module layout

Use `lib.rs` for public contracts and re-exports. Use crate-local `config.rs`, `error.rs`, `types.rs`, `metrics.rs`, `tests/`, and domain modules. Hot modules receive dependencies as traits and own their mutable state. Cold modules perform IO, config loading, process supervision, and external protocol adaptation.

## 7. File layout

```text
crates/
  hermes-wallet/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-ledger/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-treasury/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-compliance/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-events/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
```

## 8. Key traits

- `WalletService`; `LedgerService`; `JournalRepository`; `DepositProcessor`; `WithdrawalProcessor`; `TreasuryService`; `ReconciliationJob`; `StatementGenerator`.

## 9. Key structs/enums

`WalletAccount`, `WalletBalanceProjection`, `JournalEntry`, `JournalLine`, `Deposit`, `WithdrawalRequest`, `MakerCheckerApproval`, `TreasuryMovement`, `ReconciliationReport`, `AccountStatement`, `AuditEvidenceBundle`.

## 10. Error types

Use domain-specific error enums with stable reject/recovery codes, e.g. `InvalidCommand`, `ArithmeticOverflow`, `InvariantViolation`, `Duplicate`, `InsufficientAvailable`, `QueueFull`, `EventAppendFailed`, `HashChainMismatch`, `SnapshotInvalid`, `ReplayDiverged`, `ExternalDependencyUnavailable`, and `ConfigurationInvalid`. Errors must include machine-actionable codes and avoid secret/account-sensitive debug leakage.

## 11. Control flow

Wallet projections are event-derived balances; ledger is double-entry and append-only. Deposits credit clearing accounts after confirmation policy. Withdrawals create holds, pass compliance and maker-checker, call HSM/MPC boundary for signing, then post journals. Internal transfers and treasury movements use balanced journals. Insurance fund, fees, rebates, and revenue are separate ledger accounts. Reconciliation compares chain/custodian balances, wallet projections, and ledger totals. Statements and audit bundles are generated from immutable journals/events.

## 12. Rust-style pseudocode

```rust
fn credit_deposit(dep){ verify_confirmations(dep); post_journal(debit(custody), credit(user_wallet)); emit_event(DepositCredited); }
fn request_withdrawal(req){ validate_available(req); create_withdrawal_hold(req); compliance.screen(req); emit_event(WithdrawalRequested); }
fn approve_withdrawal(id, approver){ maker_checker.approve(id, approver); if quorum_met { treasury.sign_via_hsm_mpc(id); post_journal(withdrawal_lines); } }
fn post_journal(j){ assert_balanced(j); repo.append(j); projections.apply(j); }
fn project_wallet_balance(acct){ fold_events_and_journals(acct) }
fn reconcile_wallet_ledger(){ compare(sum_wallets, ledger_control_accounts, custody_balances) }
fn generate_statement(acct, range){ return StatementGenerator::from_journals(acct, range); }
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
