# Gateway Request Flow Example

This example shows a signed private REST order request flowing through `hermes-gateway`, being authenticated, authorized, rate-limited, normalized, routed to Book Core, acknowledged from a durable event, and published to private streams. The same semantic stages apply to WebSocket, FIX, OUCH, and SBE after protocol-specific adapters produce a canonical request context and order envelope.

## 1. HTTP Request

```http
POST /v1/orders HTTP/1.1
Host: api.hermesnet.example
Content-Type: application/json
X-Hermes-Api-Key: key_123
X-Hermes-Timestamp: 2026-07-10T12:00:00.000Z
X-Hermes-Nonce: nonce_abc
X-Hermes-Signature: hmac-sha256=:base64_signature:
X-Hermes-Idempotency-Key: acct_9:cli_7788

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

Gateway creates a `RequestContext` with request ID, trace ID, protocol `REST`, route `POST /v1/orders`, remote address, received timestamp, body size, and correlation metadata. Body and header sizes are checked before parsing. Financial values remain strings until the parser converts them to fixed-point integer wrappers during normalization; no floating-point value is created.

## 2. Authentication

The gateway extracts `X-Hermes-Api-Key`, timestamp, nonce, and signature, then loads `ApiKey(key_123)` from the bounded API-key cache. If the cache entry is missing or stale, a cold-path refresh may run. Disabled, expired, or unknown keys reject with `Unauthorized`.

The HMAC verifier canonicalizes method, path, sorted query, signed headers, timestamp, nonce, and raw body hash. Example canonical payload:

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

Authentication failures return `Unauthorized`, `BadSignature`, or `ClockSkew`; no authorization, idempotency record, normalization, or router submission occurs.

## 3. Authorization

The authorization service checks typed action and resource:

```text
action = PlaceOrder
resource = Account(acct_9), Symbol(BTC-USD)
```

A read-only key, wrong account, disabled symbol, missing route permission, or credential mode that cannot place orders rejects with `Forbidden` and never reaches idempotency or routing.

## 4. Rate Limiting

Gateway evaluates token-bucket and sliding-window policies for keys such as:

```text
ip:203.0.113.10
api_key:key_123
account:acct_9
route:POST /v1/orders
symbol:BTC-USD
```

If any required bucket denies the request, the gateway returns:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 1
Content-Type: application/json

{"code":"RATE_LIMITED","message":"rate limit exceeded"}
```

No idempotency record and no downstream command are created for a pre-admission rate-limit reject unless policy explicitly records rejected attempts.

## 5. Idempotency

For the first request, the idempotency key `(acct_9, POST /v1/orders, cli_7788)` transitions atomically:

```text
Vacant -> InFlight
```

Possible outcomes are:

- `Absent` or `Vacant` creates an `InFlight` ticket and continues.
- `InFlight` returns `DuplicateInFlight` or waits according to route policy.
- `TerminalAccepted` returns the prior accepted response summary.
- `TerminalRejected` returns the prior deterministic reject summary when safe.

Concurrent duplicates do not submit another command.

## 6. Normalization

The REST body is parsed into an `OrderEnvelope`, then converted to a canonical order:

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

Normalization validates domain constraints, resolves the `BookRoute` for `BTC-USD`, and wraps the `NormalizedOrder` with `RequestContext`, `AuthenticatedUser`, idempotency key, and trace context. Floating-point values are never used.

## 7. Router Submission

The router maps `BTC-USD` to a book partition:

```text
BookRoute { book_id: book_btc_usd_0, partition: 0 }
```

`RouterClient` then attempts bounded admission:

1. Check route readiness.
2. Acquire an admission permit or queue slot.
3. Submit the normalized envelope.
4. Await a downstream terminal response until the configured deadline.

The router queue is bounded. If the queue is full before admission, the gateway returns `ENGINE_BUSY` immediately and the order is not durably admitted.

## 8. Execution Report and HTTP Response

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

Gateway completes idempotency:

```text
InFlight -> Resolved(original_response)
```

Then it returns a response only after the durable ack contract is satisfied:

```http
HTTP/1.1 201 Created
Content-Type: application/json
X-Hermes-Request-Id: req_abc

{
  "status": "accepted",
  "order_id": "ord_456",
  "client_order_id": "cli_7788",
  "book_sequence": 998877,
  "event_ref": "42:8192:h_abc",
  "request_id": "req_abc"
}
```

The response contains correlation data and durable references but no secrets, signatures, internal queue names, or stack traces.

## 9. Private Stream Delivery

The execution report publisher transforms the durable event reference into an account-scoped report:

```json
{
  "type": "execution_report",
  "account_id": "acct_9",
  "order_id": "ord_456",
  "client_order_id": "cli_7788",
  "exec_type": "NEW",
  "order_status": "NEW",
  "book_id": "book_btc_usd_0",
  "book_sequence": 998877,
  "event_ref": "42:8192:h_abc"
}
```

`PrivateStreamPublisher` enqueues the report to all active private sessions for `acct_9` and to configured drop-copy sessions. For each subscribed session, the gateway verifies authorization, enqueues into a bounded outbox, emits fanout metrics, and disconnects slow clients rather than blocking report publication. Recoverable gap semantics let reconnecting clients catch up from event-derived reports.

## 10. Error Case: Bad Signature

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

No authorization, account rate-limit bucket, idempotency begin, normalization, or router submission occurs. Logs contain only safe request metadata and the redacted error code.

## 11. Error Case: Validation Reject

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

## 12. Duplicate Retry

If the client retries the same request with the same idempotency key after the original resolved, the gateway returns the stored terminal response:

```text
Resolved(original_response) -> DuplicateResolved(original_response)
```

Response:

```http
HTTP/1.1 201 Created
Content-Type: application/json
Idempotent-Replay: true
X-Hermes-Request-Id: req_retry

{
  "code": "ORDER_ACCEPTED",
  "order_id": "ord_456",
  "client_order_id": "cli_7788",
  "book_sequence": 998877,
  "event_ref": "42:8192:h_abc",
  "request_id": "req_retry"
}
```

Gateway does not submit a second command to the router. The response is reconstructed from the idempotency terminal result or durable event-derived state.

## 13. ENGINE_BUSY Case

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
X-Hermes-Request-Id: req_busy

{
  "code": "ENGINE_BUSY",
  "message": "matching engine admission is temporarily busy",
  "retryable": true,
  "request_id": "req_busy"
}
```

`ENGINE_BUSY` means this admission attempt did not create a durable order event. The client may retry with the same idempotency key. The gateway must not convert router-full overload into an unbounded queue, and the response is not replayed as an order event during recovery.

## 14. Ambiguous Router Timeout

If the gateway submitted to the router and then lost the response before knowing the terminal result, it must not blindly resubmit. It returns a pending or unknown response by route policy and reconciles from durable event/query state:

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

## 15. Replay Interpretation

During replay/recovery, HermesNet rebuilds order/report state from durable events containing `EngineEventRef { segment: 42, offset: 8192, hash: h_abc }`. Gateway-local facts such as request latency, TCP connection ID, rate-limit token balance, and WebSocket session ID are operational telemetry and are not replayed as business state.

Private stream catchup after reconnect is built from event-derived execution reports. If the replayed event stream contains the `ORDER_ACCEPTED` event for `cli_7788`, the client receives the same order status and event reference even if the original gateway process has restarted. Authentication is not re-run for historical events. `ENGINE_BUSY`, malformed, unauthorized, forbidden, and rate-limit rejects that never reached durable admission are not interpreted as order lifecycle events.

## 16. Equivalent WebSocket, FIX, OUCH, and SBE Flow

- WebSocket: an authenticated session sends an order frame; gateway applies authorization, rate limit, idempotency, normalization, router admission, and private report delivery through the same core services.
- FIX: connectivity handles logon, sequencing, resend, heartbeat, and logout; gateway maps `NewOrderSingle`, cancel, and replace into shared order envelopes and maps outcomes into execution reports or business rejects.
- OUCH: connectivity decodes low-latency binary packets; gateway authenticates the session and normalizes enter, cancel, and replace packets without JSON or floating-point conversion.
- SBE: schema/template decoding occurs at the boundary; business semantics are identical after conversion to `OrderEnvelope`.
