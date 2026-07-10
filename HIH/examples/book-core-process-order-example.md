# Book Core Process Order Example

This example shows a practical fixed-point, event-first order flow through `hermes-book`. Values are integers in configured units. No floats are used.

## Scenario

Book: `BTC-USD-PERP`.

Scales:

- price scale: cents, `1 USD = 100`;
- quantity scale: contracts, `1 contract = 1`;
- fee scale: micro-units, `1 unit = 1`.

Existing ask level before the command:

| Resting order | Side | Price | Open quantity | Account | Priority sequence |
| --- | --- | ---: | ---: | --- | ---: |
| `ORD-MAKER-77` | Sell | `30_000_00` | `7` | `ACCT-MAKER-9` | `41` |

The next book-local sequence is `42`. The previous event hash is `HASH-000041`.

## Input `OrderEnvelope`

```text
OrderEnvelope {
  command_id: CMD-9001,
  client_order_id: CL-ABC-001,
  order_id: ORD-TAKER-55,
  book_id: BTC-USD-PERP,
  account_id: ACCT-TAKER-3,
  side: Buy,
  order_type: Limit,
  price: 30_000_00,
  quantity: 10,
  time_in_force: Gtc,
  self_trade_prevention: CancelNewest
}
```

`price = 30_000_00` means `30000.00` in cents, but the implementation stores and compares only the integer `30_000_00`.

## `BookCoreCommand`

Gateway normalization has already occurred. The Book Core receives:

```text
BookCoreCommand::Place(OrderEnvelope { ... })
```

The command is already typed, bounded, authenticated upstream, and contains fixed-width identifiers plus fixed-point integers.

## Processing Stages

### 1. Poll command

The runtime polls a bounded `BookCommandSource` and receives `CMD-9001` without blocking indefinitely.

### 2. Validate command

Book Core checks:

- book ID equals `BTC-USD-PERP`;
- side is `Buy`;
- price `30_000_00` is non-zero and matches the configured price scale;
- quantity `10` is non-zero and matches the configured quantity scale;
- command payload has no unbounded fields.

No state mutation has occurred yet.

### 3. Assign book-local sequence

`SequenceAllocator` assigns:

```text
BookSequence(42)
```

This sequence is local to `BTC-USD-PERP`. No global sequencer is called.

### 4. Build `ProcessOrderContext`

```text
ProcessOrderContext {
  command_id: CMD-9001,
  book_id: BTC-USD-PERP,
  sequence: 42,
  order_id: ORD-TAKER-55,
  account_id: ACCT-TAKER-3,
  side: Buy,
  price: 30_000_00,
  quantity: 10,
  time_in_force: Gtc
}
```

The context is immutable for the rest of the pipeline.

## Risk Result

Book Core invokes `RiskBoundary` with the immutable context.

Example deterministic result:

```text
RiskDecision::Accepted {
  reservation_id: RSV-7001,
  reserved_quote: 300_000_00,
  reserved_base: 0,
  risk_version: 118
}
```

Explanation:

- taker wants to buy up to `10` contracts;
- limit price is `30_000_00` cents;
- quote reservation is `10 * 30_000_00 = 300_000_00` cents;
- risk version `118` identifies the bounded in-memory risk view used for this deterministic decision.

If this result were a reject, matching would not run and book state would not mutate.

## Matching Result

Book Core invokes `MatchingBoundary` with mutable `BookCoreState` and preallocated scratch buffers.

The buy order crosses resting ask `ORD-MAKER-77` at `30_000_00` for quantity `7`.

```text
MatchingResult {
  fills: [
    Fill {
      maker_order_id: ORD-MAKER-77,
      taker_order_id: ORD-TAKER-55,
      price: 30_000_00,
      quantity: 7,
      maker_account_id: ACCT-MAKER-9,
      taker_account_id: ACCT-TAKER-3,
      maker_priority_sequence: 41
    }
  ],
  taker_remaining_quantity: 3,
  resting_order: Some(OrderRested {
    order_id: ORD-TAKER-55,
    side: Buy,
    price: 30_000_00,
    open_quantity: 3,
    priority_sequence: 42
  }),
  removed_orders: [ORD-MAKER-77]
}
```

State effects before event append are held inside the single writer. The success is not externally visible until append completes.

## Clearing Result

Book Core invokes `ClearingBoundary` to derive immutable deltas from the fill.

```text
ClearingResult {
  deltas: [
    AccountDelta {
      account_id: ACCT-TAKER-3,
      asset: USD,
      available_delta: -210_000_00,
      reserved_delta: -210_000_00,
      position_delta: 7
    },
    AccountDelta {
      account_id: ACCT-MAKER-9,
      asset: USD,
      available_delta: 210_000_00,
      reserved_delta: 0,
      position_delta: -7
    }
  ],
  fee_deltas: [
    FeeDelta { account_id: ACCT-TAKER-3, asset: USD, amount: 600 },
    FeeDelta { account_id: ACCT-MAKER-9, asset: USD, amount: 300 }
  ]
}
```

All values are integers. The notional is `7 * 30_000_00 = 210_000_00` cents.

## EngineEvent

Book Core builds an immutable event candidate.

```text
EngineEvent {
  schema_version: 1,
  book_id: BTC-USD-PERP,
  sequence: 42,
  command_id: CMD-9001,
  event_type: OrderPartiallyFilledAndRested,
  previous_hash: HASH-000041,
  payload: {
    order_id: ORD-TAKER-55,
    account_id: ACCT-TAKER-3,
    side: Buy,
    price: 30_000_00,
    original_quantity: 10,
    filled_quantity: 7,
    resting_quantity: 3,
    fills: [...],
    clearing_deltas: [...],
    risk_reservation_id: RSV-7001
  },
  event_hash: HASH-000042
}
```

The canonical serialized bytes include `previous_hash`. After `event_hash` is computed, the event is immutable.

## Append Before Response

`EventAppender` verifies `previous_hash == HASH-000041`, appends the event, and returns:

```text
AppendedEvent {
  book_id: BTC-USD-PERP,
  sequence: 42,
  event_hash: HASH-000042,
  log_offset: 88128
}
```

Only now may Book Core publish success.

## Response

```text
BookCoreResponse::PartiallyFilled {
  command_id: CMD-9001,
  sequence: 42,
  event: AppendedEvent {
    book_id: BTC-USD-PERP,
    sequence: 42,
    event_hash: HASH-000042,
    log_offset: 88128
  },
  remaining_qty: 3
}
```

The response includes durable event identity. If the response sink fails after append, the durable event remains the source of truth.

## Replay Interpretation

During replay, Book Core does not call live risk, gateway, or response publication.

Replay steps:

1. restore snapshot with `last_sequence = 41` and `last_event_hash = HASH-000041`;
2. read event `sequence = 42`;
3. verify `previous_hash = HASH-000041`;
4. verify canonical event hash `HASH-000042`;
5. apply payload deterministically:
   - remove `ORD-MAKER-77` because its open quantity becomes `0`;
   - insert `ORD-TAKER-55` bid with open quantity `3` at price `30_000_00`;
   - record clearing deltas as event interpretation for downstream replay;
6. set state last sequence to `42` and hash head to `HASH-000042`.

The replayed state hash must equal the live state hash after the original append.

## Failure Case Example: Event Append Failure

Assume matching and clearing succeeded internally, but appender returns `EventAppendFailed` before the event is durable.

Required behaviour:

```text
BookCoreFatalError::EventAppendFailed
runtime_state = Quarantined
no success response is published
writer requires recovery from last verified snapshot and event log
```

The implementation must not publish `PartiallyFilled`. If local memory had been mutated before append, recovery determines durable truth from snapshot plus event log. Because event `42` is absent, the client cannot observe a durable success.

## Failure Case Example: Risk Reject

Risk rejects because available quote reservation is insufficient.

```text
RiskDecision::Rejected {
  code: INSUFFICIENT_AVAILABLE,
  required_quote: 300_000_00,
  available_quote: 125_000_00,
  risk_version: 119
}
```

Book Core returns:

```text
BookCoreResponse::Rejected {
  command_id: CMD-9001,
  code: INSUFFICIENT_AVAILABLE,
  sequence: Some(42)
}
```

No matching mutation occurs. If the deployment records reject events, the reject event must be appended before this response. If it does not record reject events, the rejection must remain non-mutating and deterministic.

## Duplicate Retry Example

The client sent `CMD-9001`, the event appended, and the process crashed before response publication. After recovery, the client retries the same `client_order_id = CL-ABC-001`.

Recovery has already replayed event `42` with `HASH-000042`. The duplicate detector maps `CL-ABC-001` to the durable result.

Response:

```text
BookCoreResponse::PartiallyFilled {
  command_id: CMD-9001,
  sequence: 42,
  event: AppendedEvent {
    book_id: BTC-USD-PERP,
    sequence: 42,
    event_hash: HASH-000042,
    log_offset: 88128
  },
  remaining_qty: 3
}
```

No new sequence is assigned for the duplicate durable result, and no duplicate event is appended.
