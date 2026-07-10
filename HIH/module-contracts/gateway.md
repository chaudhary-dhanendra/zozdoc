# Module Contract: Gateway

## Responsibility

The Gateway module is the HermesNet ingress and private-egress boundary. It authenticates and authorizes clients, enforces rate limits and idempotency, normalizes REST/WebSocket/FIX/OUCH/SBE requests into canonical domain commands, submits commands to bounded router queues, maps downstream outcomes into stable protocol responses, and publishes private execution reports only from durable event-derived data.

## Owned crates

- `crates/hermes-gateway/`: business ingress semantics, auth orchestration, rate limit, idempotency, normalization, router client, sessions, private streams, metrics, tracing, health, shutdown.
- `crates/hermes-connectivity/`: network/protocol adapters, REST/WebSocket/FIX/OUCH/SBE framing, TLS binding, heartbeats at transport/protocol layer, connection draining, codec tests.

## Public interfaces

- `GatewayServer`: start, readiness, liveness, drain, shutdown.
- `RestGateway`: public/private REST handling and order/cancel/replace route semantics.
- `WebSocketGateway`: session open, frame handling, private subscription, close.
- `FixGateway`: logon, business message mapping, logout.
- `OuchGateway`: login, packet mapping, disconnect.
- `RouterClient`: bounded route resolution, admission, durable ack waiting, queue depth.
- `AuthenticationService`: API-key/HMAC/JWT/session credential validation.
- `AuthorizationService`: account, route, symbol, and operation permission decisions.
- `ApiKeyStore`: key lookup, cache invalidation, revocation handling.
- `JwtVerifier`: issuer/audience/signature/claim validation.
- `HmacVerifier`: canonical request verification and replay-window checks.
- `RateLimiter`: token-bucket and sliding-window decisions.
- `IdempotencyStore`: atomic begin/complete/lookup for duplicate-safe commands.
- `SessionRegistry`: WebSocket/FIX/OUCH session lifecycle and cleanup.
- `PrivateStreamPublisher`: bounded private stream fanout.
- `ExecutionReportPublisher`: durable event-to-report publication.
- `HeartbeatService`: heartbeat observation and expiry detection.
- `HealthService`: readiness and liveness.
- `GatewayMetrics` and `GatewayTracing`: observability without secret leakage.

## Core structs
The Gateway boundary owns HermesNet external ingress and private egress. It accepts REST, WebSocket, FIX, OUCH, and SBE traffic; authenticates and authorizes principals; enforces rate limits and idempotency; normalizes requests into canonical order commands; submits commands to bounded router interfaces; maps downstream results to protocol responses; and publishes private execution reports from durable event references.

The Gateway does not own book state, matching state, clearing state, wallet balances, or the event log. It is replay-aware but not the business-state replay engine.

## Interfaces

Required public interfaces:

- `GatewayServer`: start, supervise, health, readiness, drain, shutdown.
- `RestGateway`: REST order, cancel, replace, and query handling.
- `WebSocketGateway`: handshake, frame handling, private subscriptions, disconnect.
- `FixGateway`: logon, sequence handling, application messages, logout.
- `OuchGateway`: binary login, packet handling, disconnect.
- `SbeBoundary`: schema-version negotiation and binary message adaptation.
- `AuthenticationService`: API key, JWT, HMAC, and session authentication.
- `AuthorizationService`: account/action/resource permission enforcement.
- `ApiKeyStore`: bounded key lookup/cache/refresh/rotation behavior.
- `JwtVerifier`: issuer, audience, time, algorithm, key, and signature validation.
- `HmacVerifier`: canonical request construction and constant-time signature verification.
- `RateLimiter`: token-bucket and sliding-window decisions.
- `IdempotencyStore`: begin, complete, duplicate, reconcile, and expiry behavior.
- `SessionRegistry`: bounded session insert, lookup, transition, reconnect, cleanup.
- `HeartbeatService`: ping/pong, FIX test request, timeout actions.
- `RouterClient`: bounded book-route admission and durable response wait.
- `ExecutionReportPublisher`: event-ref to execution-report construction.
- `PrivateStreamPublisher`: account-scoped bounded private fanout and drop copy.
- `HealthService`: liveness and readiness snapshots.
- `GatewayMetrics`: counters, histograms, gauges with bounded labels.
- `GatewayTracing`: trace/span creation and secret redaction.

Core data types:

- `GatewayConfig`
- `GatewayServer`
- `GatewayConnection`
- `GatewaySession`
- `ApiKey`
- `AuthenticatedUser`
- `RequestContext`
- `ResponseContext`
- `OrderEnvelope`
- `NormalizedOrder`
- `GatewayResponse`
- `GatewayError`
- `ConnectionState`
- `SessionState`
- `RateLimitDecision`
- `GatewayMetrics`

## Inputs

- REST requests, WebSocket frames, FIX messages, OUCH packets, and SBE envelopes.
- API-key, JWT, HMAC, and protocol logon credentials.
- Route maps, permission policies, rate-limit policies, idempotency records, durable event/report projections.
- Shutdown, readiness, drain, and recovery signals.

## Outputs

- Stable protocol responses and rejects.
- Canonical `NormalizedOrder` commands submitted through `RouterClient`.
- Private execution reports, drop-copy messages, and private stream frames derived from durable events.
- Metrics, traces, structured logs, health/readiness/liveness responses.

## Allowed Dependencies

Allowed dependencies:

- domain ID and fixed-point crates;
- event-reference and execution-report contracts;
- connectivity codec crate;
- router contract crate;
- injected clock/time abstractions;
- cryptographic libraries for HMAC/JWT verification;
- Tokio and HTTP/TCP/TLS/WebSocket/FIX/OUCH/SBE protocol libraries at the boundary;
- metrics/tracing backends through narrow traits;
- secret-loader traits on cold paths.

## Forbidden Dependencies and Behavior

Forbidden:

- direct dependency on book, matching, clearing, or wallet internals;
- floating-point arithmetic for price, quantity, fee, balance, or margin values;
- unbounded queues, outboxes, maps, caches, or retry buffers;
- blocking database/secret-store calls in hot protocol handlers;
- Kafka or other asynchronous log as pre-decision admission buffer;
- externally visible success before downstream durable event acknowledgement;
- retry after ambiguous downstream admission;
- secret material in logs, traces, metrics, panic messages, or protocol errors;
- accepting unknown protocol versions, JWT algorithms, SBE templates, or oversized frames;
- shared mutable book state in gateway modules.

## Invariants

- Every mutating private request is authenticated, authorized, rate-limited, idempotency-checked, normalized, and submitted at most once.
- `ENGINE_BUSY` means no durable order event was created by that gateway admission attempt.
- `DuplicateResolved` returns the original terminal response for the same idempotency key.
- `DuplicateInFlight` never causes a second downstream submission.
- Private stream messages are derived from durable event references or replay-derived reports.
- Session cleanup releases subscriptions, heartbeat state, outboxes, and rate-limit/session counters.
- Readiness is false during drain, router outage, critical queue saturation, and uninitialized auth state.
- Gateway-local sessions, rate-limit buckets, and latency metrics are operational state, not replayed business truth.

## Performance Targets

- REST signed order gateway overhead p99 under 2 ms excluding downstream book work.
- WebSocket order gateway overhead p99 under 1 ms excluding downstream book work.
- FIX/OUCH/SBE decode and normalization p99 under 500 microseconds.
- API-key cache hit p99 under 100 microseconds.
- HMAC canonicalization and verification p99 under 250 microseconds for normal payloads.
- Rate-limit decision p99 under 50 microseconds.
- Idempotency begin/complete p99 under 200 microseconds with hot cache.
- Overload rejection remains bounded under sustained router-full load.

## Security Rules

- TLS is mandatory for external listeners except local tests.
- API keys, JWTs, HMAC secrets, bearer tokens, cookies, nonce stores, and session tokens are secret.
- HMAC compare must be constant time.
- JWT verification must reject `none`, algorithm confusion, wrong issuer, wrong audience, expired claims, premature `nbf`, and unknown key ids after refresh policy.
- API-key cache must support disable, rotation, expiry, bounded TTL, and secret zeroization.
- Authorization must use typed actions and resources, not ad hoc route strings.
- Admin health/debug endpoints must be separately authorized or bound to a private listener.

## Replay Interaction

Gateway replay awareness is correlation-based:

- request id and trace id correlate gateway telemetry;
- client order id and idempotency key prevent duplicate admission;
- book id, book-local sequence, order id, and `EngineEventRef` correlate protocol responses and private reports with replayable business events;
- reconnect catchup and drop copy use event-derived reports;
- transient sessions, heartbeats, rate-limit buckets, and auth cache hits are not replayed as business state.

If recovery finds unresolved idempotency records after possible router admission, the gateway must reconcile from durable events or downstream query state before admitting a retry for the same key.

## Implementation Checklist

- [ ] Define config, errors, states, request/response contexts, order envelopes, and normalized orders.
- [ ] Implement connectivity codecs and golden-vector tests for REST/WebSocket/FIX/OUCH/SBE.
- [ ] Implement API-key cache, JWT verifier, and HMAC verifier with security tests.
- [ ] Implement authorization with account/action/resource permissions.
- [ ] Implement token-bucket and sliding-window rate limiting with bounded maps.
- [ ] Implement idempotency begin/complete/duplicate/reconcile semantics.
- [ ] Implement session registry, heartbeat, reconnect, and cleanup.
- [ ] Implement bounded router client and `ENGINE_BUSY` mapping.
- [ ] Implement REST, WebSocket, FIX, OUCH, and SBE lifecycles using shared services.
- [ ] Implement execution reports, private streams, drop copy, and slow-client eviction.
- [ ] Implement metrics, tracing, redaction, liveness, readiness, shutdown, and recovery.
- [ ] Add unit, integration, property, fuzz, restart, replay-correlation, and benchmark coverage.
