# Gateway Request Flow Example

This example shows a signed REST order request flowing through `hermes-gateway`, being normalized, routed to Book Core, acknowledged from a durable event, and published to private streams. The same logical flow applies to WebSocket, FIX, OUCH, and SBE after protocol-specific parsing.

## 1. HTTP Request

```http
POST /v1/orders HTTP/1.1
Host: api.hermesnet.example
Content-Type: application/json
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
