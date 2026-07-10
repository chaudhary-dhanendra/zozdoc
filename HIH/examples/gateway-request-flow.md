# Gateway Request Flow Example

This example shows a private REST order request moving through the HermesNet Gateway. The same semantic stages apply to WebSocket, FIX, OUCH, and SBE after their protocol adapters produce a canonical request context and order envelope.

## 1. HTTP request
This example shows a signed REST order request flowing through `hermes-gateway`, being normalized, routed to Book Core, acknowledged from a durable event, and published to private streams. The same logical flow applies to WebSocket, FIX, OUCH, and SBE after protocol-specific parsing.

## 1. HTTP Request

```http
POST /v1/orders HTTP/1.1
Host: api.hermesnet.example
Content-Type: application/json
X-Hermes-Api-Key: key_live_123
X-Hermes-Timestamp: 2026-07-10T12:00:00Z
X-Hermes-Nonce: 01J2EXAMPLE
X-Hermes-Signature: hmac-sha256=:redacted:
X-Hermes-Idempotency-Key: acctA:client-order-42

{
  "account_id": "acctA",
  "symbol": "BTC-USD",
  "side": "buy",
  "type": "limit",
  "quantity": "0.25000000",
  "price": "65000.00",
  "time_in_force": "GTC",
  "client_order_id": "client-order-42"
}
```

Gateway creates a `RequestContext` with request ID, trace ID, protocol `REST`, route `POST /v1/orders`, remote address, received timestamp, body size, and correlation metadata. Body and header sizes are checked before parsing.

## 2. Authentication

1. Extract API key ID, timestamp, nonce, and signature.
2. Look up `ApiKey` in bounded API-key cache.
3. Validate key status, expiry, allowed source policy, and account scope.
4. Build canonical request from method, path, sorted query, signed headers, timestamp, nonce, and raw body hash.
5. Verify HMAC with constant-time comparison.
6. Produce `AuthenticatedUser` for account `acctA`, auth scheme `ApiKeyHmac`, permission set, and policy version.

Authentication failures return `Unauthorized`, `BadSignature`, or `ClockSkew`; no idempotency record or router submission is created.

## 3. Authorization

Gateway checks that the authenticated user can perform `orders:create` for account `acctA` and symbol `BTC-USD`. A read-only key, wrong account, disabled symbol, or missing route permission returns `Forbidden` with stable client-facing wording.

## 4. Rate limit

Gateway evaluates token-bucket and sliding-window policies for keys such as:

- remote IP + route,
- API key + route,
- account + route,
- account + symbol,
- session or connection where applicable.

If denied, the response is a stable `RateLimited` error with safe retry metadata when configured. No idempotency record or router submission is created.

## 5. Idempotency

For `acctA:orders:create:client-order-42`, the idempotency store performs atomic `begin`:

- `Absent` → create `InFlight` ticket and continue.
- `InFlight` → return `DuplicateInFlight` or wait according to route policy.
- `TerminalAccepted` → return the prior accepted response summary.
- `TerminalRejected` → return the prior deterministic reject summary when safe.

## 6. Normalization

The REST body is parsed into `OrderEnvelope`, then normalized:

```text
OrderEnvelope
  account_id: acctA
  symbol: BTC-USD
  side: buy
  type: limit
  quantity: "0.25000000"
  price: "65000.00"
  tif: GTC
  client_order_id: client-order-42
```

Normalization converts decimal strings to fixed-point integer wrappers, validates domain constraints, resolves the `BookRoute` for `BTC-USD`, and creates `NormalizedOrder`. Floating-point values are never used.

## 7. Book routing

`RouterClient` resolves `BTC-USD` to the responsible book route and attempts bounded admission:

1. Check route readiness.
2. Acquire admission permit or queue slot.
3. Submit `NormalizedOrder` and `RequestContext`.
4. Await durable book/risk response until configured deadline.

A full route queue returns `ENGINE_BUSY` immediately and the order is not durably admitted.

## 8. Book response

Example accepted downstream response:

```json
{
  "kind": "DurableAccepted",
  "order_id": "ord_789",
  "client_order_id": "client-order-42",
  "event_ref": {
    "partition": 7,
    "sequence": 991122,
    "hash": "event_hash_redacted"
  }
}
```

Gateway completes the idempotency ticket as `TerminalAccepted` and builds the REST response only after the durable ack contract is satisfied.

## 9. HTTP success response
X-Hermes-Api-Key: key_123
X-Hermes-Timestamp: 2026-07-10T12:00:00.000Z
X-Hermes-Nonce: nonce_abc
X-Hermes-Signature: hmac-sha256=:base64_signature:
Idempotency-Key: acct_9:cli_7788

{
  "account_id": "acct_9",
  "symbol": "BTC-USD",
  "side": "BUY",
  "type": "LIMIT",
  "time_in_force": "GTC",
  "quantity": "0.25000000",
  "price": "64000.00",
  "client_order_id": "cli_7788"
}
```

Financial values are strings. The parser converts them to fixed-point integer wrappers during normalization. No floating-point value is created.

## 2. Authentication

The gateway builds a `RequestContext`, extracts `X-Hermes-Api-Key`, and loads `ApiKey(key_123)` from the bounded API-key cache. If the cache entry is missing or stale, a cold-path refresh may run. Disabled, expired, or unknown keys reject with `Unauthorized`.

The HMAC verifier canonicalizes:

```text
POST
/v1/orders

key_123
2026-07-10T12:00:00.000Z
nonce_abc
sha256(body)
v1
```

It checks timestamp skew, nonce replay, body hash, and signature using constant-time comparison. On success the gateway builds:

```text
AuthenticatedUser {
  account_id: acct_9,
  key_id: key_123,
  permissions: [PlaceOrder:BTC-USD, ReadOwnOrders, SubscribePrivateStream]
}
```

## 3. Authorization

The authorization service checks typed action and resource:

```text
action = PlaceOrder
resource = Account(acct_9), Symbol(BTC-USD)
```

If `key_123` is read-only or belongs to a different account, the request rejects with `Forbidden` and never reaches idempotency or routing.

## 4. Rate Limit

The gateway derives rate-limit keys:

```text
ip:203.0.113.10
api_key:key_123
account:acct_9
route:POST /v1/orders
```

It checks token-bucket burst limits and sliding-window ceilings. If any required bucket denies the request, the gateway returns:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 1
Content-Type: application/json

{"code":"RATE_LIMITED","message":"rate limit exceeded"}
```

No idempotency record and no downstream command are created for a pre-admission rate-limit reject unless policy explicitly records rejected attempts.

## 5. Idempotency

For the first request, the idempotency key `(acct_9, POST /v1/orders, cli_7788)` transitions:

```text
Vacant -> InFlight
```

Concurrent duplicates receive `DuplicateInFlight` or wait according to route policy. They do not submit another command.

## 6. Normalization

The REST body is converted to a canonical order:

```text
NormalizedOrder {
  account_id: acct_9,
  symbol: BTC-USD,
  side: Buy,
  order_type: Limit,
  time_in_force: Gtc,
  quantity: FixedQty(25_000_000),
  price: Some(FixedPrice(6_400_000)),
  client_order_id: cli_7788
}
```

The gateway wraps it in an `OrderEnvelope` with `RequestContext`, `AuthenticatedUser`, idempotency key, and trace context.

## 7. Book Routing

The router maps `BTC-USD` to a book partition:

```text
BookRoute { book_id: book_btc_usd_0, partition: 0 }
```

The gateway calls `RouterClient::try_submit(envelope)`. The router queue is bounded. If admission succeeds, the gateway waits for a downstream terminal response. If the queue is full before admission, the gateway returns `ENGINE_BUSY`.

## 8. Book Response

A successful downstream response contains a durable event reference:

```text
BookResponse::Accepted {
  order_id: ord_456,
  book_id: book_btc_usd_0,
  book_sequence: 998877,
  event_ref: EngineEventRef { segment: 42, offset: 8192, hash: h_abc },
  status: New
}
```

The gateway completes idempotency:

```text
InFlight -> Resolved(original_response)
```

Then it returns:

```http
HTTP/1.1 201 Created
Content-Type: application/json
X-Hermes-Request-Id: req_abc

{
  "status": "accepted",
  "order_id": "ord_789",
  "client_order_id": "client-order-42",
  "event_sequence": 991122,
  "request_id": "req_abc"
}
```

The response contains correlation data and durable references but no secrets, signatures, internal queue names, or stack traces.

## 10. Execution report

The execution report publisher observes the durable event/report projection and builds a private report:

{
  "code": "ORDER_ACCEPTED",
  "order_id": "ord_456",
  "client_order_id": "cli_7788",
  "book_id": "book_btc_usd_0",
  "book_sequence": 998877,
  "event_ref": "42:8192:h_abc"
}
```

## 9. Execution Report and Private Stream

The execution report publisher transforms the durable event reference into an account-scoped report:

```json
{
  "type": "execution_report",
  "account_id": "acctA",
  "order_id": "ord_789",
  "client_order_id": "client-order-42",
  "order_status": "new",
  "event_sequence": 991122
}
```

Gateway publishes this to WebSocket private streams and FIX drop-copy sessions subscribed for `acctA`. Reports are derived from durable events, not optimistic gateway state.

## 11. Private stream delivery

For each subscribed session:

1. Verify the session is active and authorized for the account.
2. Enqueue report into the bounded outbox.
3. If the outbox is full beyond policy, mark the client slow and close/disconnect with recoverable gap semantics.
4. Emit metrics for fanout latency and outbox depth.

## 12. Error case: bad signature

If the signature does not match the canonical request:

```http
HTTP/1.1 401 Unauthorized
Content-Type: application/json
X-Hermes-Request-Id: req_bad_sig

{
  "error": {
    "code": "BAD_SIGNATURE",
    "message": "request authentication failed"
  },
  "request_id": "req_bad_sig"
}
```

No authorization, rate-limit account bucket, idempotency begin, normalization, or router submission occurs. Logs contain only safe request metadata and the redacted error code.

## 13. Duplicate retry

If the client retries the exact request with the same idempotency key after the first request completed:

```http
HTTP/1.1 200 OK
Content-Type: application/json
X-Hermes-Request-Id: req_retry

{
  "status": "duplicate_resolved",
  "original_status": "accepted",
  "order_id": "ord_789",
  "client_order_id": "client-order-42",
  "event_sequence": 991122,
  "request_id": "req_retry"
}
```

Gateway does not submit a second command to the router. The response is reconstructed from the idempotency terminal result or durable event-derived state.

## 14. `ENGINE_BUSY` case

If the target book route queue is full before admission:

```http
HTTP/1.1 503 Service Unavailable
Content-Type: application/json
Retry-After: 1
X-Hermes-Request-Id: req_busy

{
  "error": {
    "code": "ENGINE_BUSY",
    "message": "matching engine is temporarily busy"
  },
  "request_id": "req_busy"
}
```

Interpretation:

- The gateway did not create a durable order event for this admission attempt.
- The client may retry using the same client order ID/idempotency key.
- Gateway metrics increment route busy counters and queue-depth gauges.
- The response is not replayed as an order event during recovery.

## 15. Replay interpretation

During replay/recovery, HermesNet rebuilds order/report state from durable events. The gateway uses rebuilt idempotency and report positions to answer duplicate retries and reconnecting private streams. Authentication is not re-run for historical events. `ENGINE_BUSY`, malformed, unauthorized, forbidden, and rate-limit rejects that never reached durable admission are not interpreted as order lifecycle events.

## 16. Equivalent WebSocket/FIX/OUCH flow

- WebSocket: authenticated session sends an order frame; gateway applies authorization, rate limit, idempotency, normalization, router admission, and private report delivery through the same core services.
- FIX: connectivity handles logon, sequencing, resend, heartbeat, and logout; gateway maps `NewOrderSingle`, cancel, and replace into shared order envelopes and maps outcomes into execution reports/business rejects.
- OUCH: connectivity decodes low-latency binary packets; gateway authenticates the session and normalizes enter/cancel/replace packets without JSON or floating-point conversion.
- SBE: schema/template decoding occurs at the boundary; business semantics are identical after conversion to `OrderEnvelope`.
  "account_id": "acct_9",
  "order_id": "ord_456",
  "client_order_id": "cli_7788",
  "exec_type": "NEW",
  "order_status": "NEW",
  "event_ref": "42:8192:h_abc"
}
```

`PrivateStreamPublisher` enqueues the report to all active private sessions for `acct_9` and to configured drop-copy sessions. Per-session outboxes are bounded. If a WebSocket client is too slow, the gateway disconnects it rather than blocking report publication.

## 10. Error Case: Validation Reject

If the request uses an invalid quantity:

```json
{"quantity":"0.000000001"}
```

and the symbol scale supports only eight decimal places, normalization fails before router admission:

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{"code":"MALFORMED_REQUEST","message":"invalid quantity scale"}
```

No durable event is required for a gateway parser reject, and no private execution report is published.

## 11. Duplicate Retry

If the client retries the same request with the same idempotency key after the original resolved, the gateway returns the stored terminal response:

```text
Resolved(original_response) -> DuplicateResolved(original_response)
```

Response:

```http
HTTP/1.1 201 Created
Content-Type: application/json
Idempotent-Replay: true

{
  "code": "ORDER_ACCEPTED",
  "order_id": "ord_456",
  "client_order_id": "cli_7788",
  "book_sequence": 998877,
  "event_ref": "42:8192:h_abc"
}
```

No second router submission occurs.

## 12. ENGINE_BUSY Case

If the router queue for `book_btc_usd_0` is full before admission:

```text
RouterClient::try_submit -> QueueFull
GatewayError::EngineBusy
```

Response:

```http
HTTP/1.1 503 Service Unavailable
Retry-After: 1
Content-Type: application/json

{
  "code": "ENGINE_BUSY",
  "message": "matching engine admission is temporarily busy",
  "retryable": true
}
```

`ENGINE_BUSY` means this admission attempt did not create a durable order event. The client may retry with the same idempotency key. The gateway must not convert router-full overload into an unbounded queue.

## 13. Ambiguous Router Timeout

If the gateway submitted to the router and then lost the response before knowing the terminal result, it must not blindly resubmit. It returns a pending/unknown response by route policy and reconciles from durable event/query state:

```http
HTTP/1.1 504 Gateway Timeout
Content-Type: application/json

{
  "code": "ORDER_STATUS_UNKNOWN",
  "client_order_id": "cli_7788",
  "message": "query order status before retrying with a new client order id"
}
```

A retry with the same idempotency key waits for reconciliation or returns the reconciled terminal response.

## 14. Replay Interpretation

During replay, the authoritative business outcome is the durable event stream containing `EngineEventRef { segment: 42, offset: 8192, hash: h_abc }`. Gateway-local facts such as request latency, TCP connection id, rate-limit token balance, and WebSocket session id are operational telemetry and are not replayed as business state.

Private stream catchup after reconnect is built from event-derived execution reports. If the replayed event stream contains the `ORDER_ACCEPTED` event for `cli_7788`, the client receives the same order status and event reference even if the original gateway process has restarted.
