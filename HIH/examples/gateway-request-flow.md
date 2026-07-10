# Gateway Request Flow Example

This example shows a private REST order request moving through the HermesNet Gateway. The same semantic stages apply to WebSocket, FIX, OUCH, and SBE after their protocol adapters produce a canonical request context and order envelope.

## 1. HTTP request

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
