# Gateway Implementation

## 1. Purpose

This volume defines how to implement gateway implementation for HermesNet in Rust. It converts HES behavior into crate boundaries, module contracts, deterministic control flow, pseudocode, tests, and benchmark expectations without creating production source files.

## 2. Related HES volumes

- HES matching, risk, settlement, wallet, derivatives, market-data, connectivity, event-log, replay, observability, security, and operations volumes as applicable to this implementation area.
- HES requirements for deterministic replay, fixed-point arithmetic, auditable financial mutation, bounded queues, and event persistence before success remain authoritative.

## HES invariants applied in this volume

- Rust owns the performance-sensitive path; adapters stay at explicit boundaries.
- All prices, quantities, fees, rates, balances, and margins use fixed-point integer wrappers; floating point is forbidden in matching, risk, settlement, replay, and certification.
- Bounded queues are mandatory; overload is reported explicitly instead of allocating without limit.
- EngineEvents are immutable after construction, persisted in a hash-chained append-only log, and appended before externally visible success.
- Replay must rebuild equivalent state from snapshots plus events, with deterministic ordering and no hidden wall-clock dependency.

## 3. Implementation scope

Includes REST, signed REST, WebSocket, FIX/OUCH/SBE ingress boundaries, validation, authentication, authorization, rate limiting, idempotency, normalized order routing, private streams, overload handling, and gateway observability/security.

## 4. Non-goals

- Do not implement production Rust source in this handbook.
- Do not redefine HES externally observable semantics.
- Do not introduce floats, unbounded queues, hot-path databases, Kafka, or shared mutable book state.
- Do not use the handbook as a substitute for conformance tests or security review.

## 5. Target crates

- `hermes-gateway`
- `hermes-connectivity`
- `hermes-auth`
- `hermes-rate-limit`
- `hermes-domain`
- `hermes-events`

## 6. Module layout

Each target crate should expose a small `lib.rs` that re-exports stable contracts and keeps implementations in modules such as `config`, `error`, `types`, `service`, `store`, `runtime`, `validation`, `metrics`, `tests`, and crate-specific modules listed below. Hot-path modules must not depend on network, SQL, async runtimes, JSON, or wall-clock APIs except through narrow cold-path adapters.

## 7. File layout

```text
crates/
  hermes-gateway/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-connectivity/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-auth/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-rate-limit/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-domain/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
  hermes-events/src/{lib.rs,config.rs,error.rs,types.rs,metrics.rs}
```

## 8. Key traits

- `GatewayService`: REST/WebSocket/FIX entry point for commands and queries.
- `AuthService`: loads API keys, validates JWT/session state, and resolves account permissions.
- `SignatureVerifier`: verifies HMAC canonical requests with timestamp and nonce windows.
- `RateLimiter`: enforces per-key, per-account, per-IP, and route buckets.
- `IdempotencyStore`: maps `(account_id, client_order_id)` to accepted/rejected terminal result.
- `OrderNormalizer`: converts REST/FIX/OUCH/SBE inputs into canonical domain commands.
- `RouterClient`: bounded admission client to book-local router queues.
- `PrivateStreamPublisher`: emits private execution reports after events are durable.
- `SessionRegistry`: tracks WebSocket/FIX sessions, subscriptions, heartbeats, and cleanup.

## 9. Key structs/enums

Define `RestRequestContext`, `AuthenticatedPrincipal`, `ApiKeyRecord`, `JwtClaims`, `GatewayOrderRequest`, `NormalizedOrderCommand`, `GatewayResponse`, `GatewayRejectCode`, `WsSession`, `RateLimitDecision`, and `IdempotencyRecord`.

## 10. Error types

Use `GatewayError` variants: `MalformedRequest`, `Unauthorized`, `Forbidden`, `BadSignature`, `ClockSkew`, `RateLimited`, `DuplicateInFlight`, `DuplicateResolved`, `EngineBusy`, `QueueFull`, `SlowClient`, `SessionClosed`, `InternalColdPathFailure`. Map errors to stable REST/FIX/WebSocket reject codes.

## 11. Control flow

REST handlers parse and validate shape, authenticate, authorize, verify signatures for private routes, enforce rate limits, check idempotency, normalize orders, attempt bounded router admission, and return only after the Book Core has appended the corresponding event or rejected before admission. WebSocket lifecycle authenticates, registers subscriptions, sends private reports from event-derived projections, and disconnects slow clients whose bounded outbound queues exceed policy. FIX/OUCH/SBE are boundary adapters that normalize to the same domain commands.

## 12. Rust-style pseudocode

```rust
fn handle_place_order(req, ctx) -> GatewayResult<Response> {
    let principal = auth.authenticate(req.headers)?;
    verify_signature(req, principal.api_key)?;
    enforce_rate_limit(&principal, "place_order")?;
    let idem = idempotency.begin(principal.account_id, req.client_order_id)?;
    let cmd = normalize_order(req.body, &principal)?;
    let accepted = submit_to_router(cmd).map_err(|e| map_busy(e))?;
    idempotency.complete(idem, accepted.summary);
    publish_private_execution_report(&accepted.event_ref);
    Ok(Response::accepted(accepted))
}
fn verify_signature(req, key) -> Result<(), GatewayError> { canonicalize_method_path_query_body(req); check_timestamp_window(req); hmac_sha256_constant_time(key.secret, canonical); }
fn enforce_rate_limit(principal, route) -> Result<(), GatewayError> { if limiter.try_take(principal, route).allowed { Ok(()) } else { Err(RateLimited) } }
fn normalize_order(input, principal) -> Result<NormalizedOrderCommand, GatewayError> { validate_symbol_side_qty_price_tif(input)?; require_permission(principal,"trade")?; convert_decimal_strings_to_fixed(input) }
fn submit_to_router(cmd) -> Result<BookAck, GatewayError> { router.try_send(cmd).map_full_to(ENGINE_BUSY)?.await_durable_ack() }
fn publish_private_execution_report(event_ref) { private_stream.enqueue_after_durable_event(event_ref); }
fn handle_websocket_connect(handshake) -> Result<SessionId, GatewayError> { let p=auth.authenticate_ws(handshake)?; limiter.try_register_session(p)?; sessions.insert(WsSession::new(p, bounded_outbox)); }
fn cleanup_session(id) { sessions.remove(id); limiter.release_session(id); private_stream.unsubscribe_all(id); zeroize_session_tokens(id); }
```

## 13. Configuration

Configure listen addresses, TLS, HMAC timestamp window, JWT issuers, API-key cache TTL, rate buckets, max request body, bounded queue sizes, WebSocket heartbeat, slow-client thresholds, and FIX/SBE codec versions. Secrets come only from approved secret loaders, never static config files.

## 14. Observability hooks

Metrics: request latency by route/outcome, signature failures, auth cache hit ratio, rate-limit decisions, idempotency hits, router queue depth, ENGINE_BUSY rejects, WebSocket sessions, slow-client disconnects. Traces must carry request_id/client_order_id/order_id without leaking secrets.

## 15. Failure handling

Reject before admission on malformed/auth/rate-limit failures. If router queue is full, return stable `ENGINE_BUSY` and do not create a durable order event. If client retries with the same `client_order_id`, return the stored terminal decision. Disconnect slow clients after bounded drops and cleanup all session resources.

## 16. Testing strategy

Unit-test canonical signing, JWT validation, permission checks, decimal-to-fixed parsing, rate buckets, idempotency races, and WebSocket cleanup. Integration-test REST/FIX/OUCH/SBE normalization equivalence and event-before-success behavior.

## 17. Benchmark strategy

Benchmark signed REST throughput, p99 auth/cache path, order normalization allocations, router admission latency, WebSocket fanout, slow-consumer eviction, and overload rejection rate.

## 18. Codex implementation tasks

1. Define gateway contracts and error mapping.
2. Implement auth/key cache and signature verifier.
3. Implement rate limiter and idempotency store.
4. Implement adapters and normalization.
5. Wire bounded router and private streams.
6. Add load/security tests.

## 19. Review checklist

- [ ] No secret material in logs.
- [ ] All private routes require auth and authorization.
- [ ] Same `client_order_id` is retry-safe.
- [ ] Bounded queues return `ENGINE_BUSY`.
- [ ] Private reports originate from durable events.

## 20. Implementation completion criteria

Complete when REST, WebSocket, FIX/OUCH/SBE boundaries share canonical validation, overload behavior is deterministic, security tests pass, and gateway outputs can be replay-correlated to EngineEvents.
