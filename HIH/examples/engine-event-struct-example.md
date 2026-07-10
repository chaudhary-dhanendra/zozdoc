# Engine Event Struct Example

This example is illustrative. Hashes, ids, account ids, and amounts are readable examples rather than production vectors. The structure shows how an incoming trade becomes a clearing result, wallet delta, ledger delta, serialized `EngineEvent`, replay input, and audit artifact.

## Incoming trade fact

```text
book_id: BTC-USD-SPOT
book_sequence: 42
command_id: cmd-2026-07-10-000001
fill_id: fill-42-0001
price: 50_000.00 USD
quantity: 0.10000000 BTC
notional: 5_000.00 USD
maker: acct-maker-7, side = Sell
taker: acct-taker-9, side = Buy
fee_schedule_version: spot-fees-v3
maker_rate: -0.0001     # rebate
taker_rate:  0.0010     # fee
rounding: half-up to USD cents
```

## Clearing result

```text
ClearingDelta {
  fill_id: fill-42-0001,
  book_id: BTC-USD-SPOT,
  product_id: BTC-USD,
  buyer: acct-taker-9,
  seller: acct-maker-7,
  base_asset: BTC,
  quote_asset: USD,
  quantity: 0.10000000 BTC,
  price: 50_000.00 USD,
  notional: 5_000.00 USD,
  maker_side: Sell,
  taker_side: Buy
}
```

## Fee calculation

```text
maker rebate = 5_000.00 * 0.0001 = 0.50 USD credit to maker
taker fee    = 5_000.00 * 0.0010 = 5.00 USD debit from taker
fee_currency = USD
```

```text
FeeBreakdown {
  schedule_version: spot-fees-v3,
  maker_fee: -0.50 USD,
  taker_fee:  5.00 USD,
  maker_rebate: 0.50 USD,
  fee_currency: USD,
  rounding_mode: HalfUpCents,
  rounding_residue: 0
}
```

## Wallet delta

```text
WalletDelta[0] taker USD available: -5_005.00, held: +0.00, reason: TradeQuoteDebitAndFee
WalletDelta[1] taker BTC available: +0.10000000, held: +0.00000000, reason: TradeBaseCredit
WalletDelta[2] maker BTC available: -0.10000000, held: +0.10000000 release/consume, reason: TradeBaseDebit
WalletDelta[3] maker USD available: +5_000.50, held: +0.00, reason: TradeQuoteCreditAndRebate
WalletDelta[4] fee-revenue USD available: +4.50, held: +0.00, reason: NetFeeRevenue
```

The net fee revenue is taker fee minus maker rebate: `5.00 - 0.50 = 4.50 USD`.

## Ledger delta

```text
LedgerDelta[0] debit  customer:taker:USD        5_005.00 USD
LedgerDelta[1] credit customer:taker:BTC        0.10000000 BTC
LedgerDelta[2] debit  customer:maker:BTC        0.10000000 BTC
LedgerDelta[3] credit customer:maker:USD        5_000.50 USD
LedgerDelta[4] credit revenue:trading-fees:USD  4.50 USD
LedgerDelta[5] debit  clearing:trade:USD        5_005.00 USD
LedgerDelta[6] credit clearing:trade:USD        5_005.00 USD
LedgerDelta[7] debit  clearing:trade:BTC        0.10000000 BTC
LedgerDelta[8] credit clearing:trade:BTC        0.10000000 BTC
```

A production ledger schema may split or name clearing accounts differently, but the journal must balance per asset and preserve fee/rebate auditability.

## Settlement journal

```text
SettlementJournal {
  journal_id: journal-BTC-USD-SPOT-42-fill-0001,
  source_event_id: event-BTC-USD-SPOT-42,
  book_id: BTC-USD-SPOT,
  book_sequence: 42,
  wallet_deltas: [5 wallet lines],
  ledger_deltas: [9 ledger lines],
  fee_breakdown: spot-fees-v3 / taker 5.00 USD / maker rebate 0.50 USD,
  balance_proof: sha256:proof_5f3a_illustrative,
  status: Ready
}
```

## EngineEvent structure

```text
EngineEvent {
  header: EngineEventHeader {
    version: 1.0.0,
    event_kind: ClearingResult,
    book_id: BTC-USD-SPOT,
    book_sequence: 42,
    global_sequence: 900142,
    event_id: event-BTC-USD-SPOT-42,
    command_id: cmd-2026-07-10-000001,
    causation_id: fill-42-0001,
    correlation_id: order-flow-abc123,
    engine_timestamp: 2026-07-10T00:00:00.000000042Z deterministic-engine-time,
    prev_hash: h_00000041_9ad2_illustrative,
    payload_hash: h_payload_42_6c01_illustrative,
    event_hash: h_event_42_50ab_illustrative
  },
  payload: EngineEventPayload::ClearingResult {
    clearing_delta: ...,
    fee_breakdown: ...,
    wallet_deltas: ...,
    ledger_deltas: ...,
    settlement_journal: ...
  }
}
```

## Serialized EngineEvent

Canonical bytes are not JSON. The following readable layout shows the schema order used to produce bytes:

```text
magic: HMEV
schema_version: 1.0.0
header.event_kind: 04
header.book_id: BTC-USD-SPOT
header.book_sequence: 42
header.global_sequence: 900142
header.event_id: event-BTC-USD-SPOT-42
header.command_id: cmd-2026-07-10-000001
header.causation_id: fill-42-0001
header.correlation_id: order-flow-abc123
header.engine_timestamp_nanos: 42
header.prev_hash: h_00000041_9ad2_illustrative
header.payload_hash: h_payload_42_6c01_illustrative
payload.discriminant: ClearingResult
payload.clearing_delta: schema-ordered fixed-width fields
payload.fee_breakdown: schema-ordered fixed-width fields
payload.wallet_delta_count: 5
payload.wallet_deltas: schema-ordered bounded vector
payload.ledger_delta_count: 9
payload.ledger_deltas: schema-ordered bounded vector
payload.settlement_journal: schema-ordered fields
header.event_hash: h_event_42_50ab_illustrative
record.length: 4096 illustrative
record.checksum: crc32c_31aa_illustrative
```

## Hash values

```text
prev_hash    = h_00000041_9ad2_illustrative
payload_hash = hash(canonical_payload_bytes)
             = h_payload_42_6c01_illustrative
event_hash   = hash(domain || version || book_id || sequence || prev_hash || payload_hash || canonical_event_without_current_hash)
             = h_event_42_50ab_illustrative
next event prev_hash must equal h_event_42_50ab_illustrative
```

## Replay interpretation

Replay verifies record framing, checksum, schema version, `payload_hash`, `event_hash`, and `prev_hash`. It then applies the stored payload exactly:

1. mark fill `fill-42-0001` as executed;
2. apply wallet deltas to wallet projection;
3. apply ledger deltas to ledger projection;
4. mark settlement journal `journal-BTC-USD-SPOT-42-fill-0001` as ready;
5. advance book sequence to `42` and hash-chain head to `h_event_42_50ab_illustrative`.

Replay does not look up the current fee schedule. The historical `5.00 USD` taker fee and `0.50 USD` maker rebate are authoritative.

## Audit interpretation

An auditor can prove:

- the trade used book-local sequence `42`;
- the event follows `h_00000041_9ad2_illustrative` and precedes any event whose `prev_hash` is `h_event_42_50ab_illustrative`;
- the taker paid `5_005.00 USD` and received `0.10000000 BTC`;
- the maker delivered `0.10000000 BTC` and received `5_000.50 USD`;
- fee revenue is `4.50 USD` after maker rebate;
- wallet and ledger interpretations are present in the immutable event;
- any correction must appear as a later compensating event rather than an edit to this event.
