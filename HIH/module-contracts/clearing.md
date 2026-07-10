# Module Contract: clearing

## Responsibility

`hermes-clearing` converts deterministic engine execution facts into auditable clearing artifacts: `ClearingDelta`, `FeeBreakdown`, `WalletDelta`, `LedgerDelta`, `SettlementJournal`, and `ClearingResult`. It is pure domain logic and must not perform durable IO or external publication.

## Interfaces

### Public traits

```rust
trait ClearingEngine {
    fn clear_execution(&mut self, input: ClearingInput<'_>) -> Result<ClearingResult, ClearingError>;
    fn clear_rejection(&mut self, input: RejectionInput<'_>) -> Result<ClearingResult, ClearingError>;
}

trait FeeCalculator {
    fn calculate_fees(&self, fill: &FillFact, schedule: FeeScheduleRef) -> Result<FeeBreakdown, ClearingError>;
}

trait WalletDeltaBuilder {
    fn build_wallet_delta(&self, delta: &ClearingDelta, fees: &FeeBreakdown) -> Result<Vec<WalletDelta>, ClearingError>;
}

trait LedgerDeltaBuilder {
    fn build_ledger_delta(&self, delta: &ClearingDelta, fees: &FeeBreakdown) -> Result<Vec<LedgerDelta>, ClearingError>;
}

trait SettlementBuilder {
    fn build_settlement(&self, delta: &ClearingDelta) -> Result<SettlementJournal, ClearingError>;
}

trait JournalBuilder {
    fn build_settlement_journal(
        &self,
        wallet: Vec<WalletDelta>,
        ledger: Vec<LedgerDelta>,
    ) -> Result<SettlementJournal, ClearingError>;
}
```

### Core structs

- `ClearingDelta`: fill id, book id, product id, buyer, seller, assets, quantity, price, notional, maker side, taker side.
- `FeeBreakdown`: schedule version, maker fee, taker fee, rebate, fee currency, rounding mode, rounding residue.
- `WalletDelta`: account id, wallet id, asset id, available delta, held delta, reason, source event id, sequence.
- `LedgerDelta`: ledger account id, asset id, debit, credit, posting type, source event id, line id.
- `SettlementJournal`: journal id, source event id, book sequence, wallet deltas, ledger deltas, fee breakdown, balance proof.
- `ClearingResult`: deterministic result id, clearing delta, fees, wallet deltas, ledger deltas, journal, status.
- `ClearingError`: stable error code plus safe message for overflow, invalid fee schedule, invalid role, unsupported product, unbalanced journal, and invariant violation.

## Invariants

- All financial arithmetic uses checked fixed-point integer types.
- No float values are accepted or produced.
- Maker/taker role comes from matching facts.
- Fee schedule version is part of the result and event payload.
- Wallet deltas explicitly state available and held changes.
- Ledger lines are double-entry and balance per asset within a journal.
- Clearing results are immutable after event construction.
- Corrections are represented by compensating events, never mutation of historical results.

## Allowed dependencies

- Hermes fixed-point and identifier crates.
- Hermes domain type crates that do not introduce IO.
- Local config/error/metrics traits.
- `hermes-events` types only at the boundary where clearing output is embedded into an event payload, if dependency direction is approved by the crate map.

## Forbidden dependencies

- SQL or database clients.
- Kafka or message brokers.
- Filesystem appenders.
- Network clients.
- Wall-clock reads during deterministic processing.
- Random number generation.
- Floating-point math.
- Unbounded queues or unbounded vector construction in hot paths.

## Replay guarantees

Clearing replay uses the amounts stored in `EngineEvent` payloads. It must not recalculate historical fees using current schedules. Replaying a committed clearing result must reproduce identical wallet deltas, ledger deltas, journal balance proofs, and result status.

## Audit guarantees

Every clearing mutation must be traceable to a source event id, book id, book sequence, fee schedule version, and deterministic journal id. A journal must provide enough data for audit without querying mutable fee configuration.

## Benchmark expectations

- Fee calculation latency and overflow path coverage.
- Wallet and ledger delta construction latency by fill count.
- Settlement journal generation latency and allocation count.
- Balance validation throughput.
- No accidental allocation regressions in hot-path fee calculation.

## HES references

Relevant HES areas: clearing, settlement, events, replay, wallet/ledger, fixed-point arithmetic, observability, and operational recovery. HES principles enforced here include immutable events, append-before-success, deterministic replay, auditable financial mutation, and compensating-event reversal.

## Implementation checklist

- [ ] Implement fixed-point fee calculator with maker/taker/rebate tests.
- [ ] Implement wallet delta builder with held/available invariants.
- [ ] Implement ledger delta builder with debit/credit balance checks.
- [ ] Implement settlement journal builder with deterministic ids.
- [ ] Add rejection handling with no unauthorized balance mutation.
- [ ] Add property tests for balance conservation and deterministic results.
- [ ] Add golden clearing examples used by event serialization tests.
- [ ] Confirm forbidden dependencies are absent from the crate.
