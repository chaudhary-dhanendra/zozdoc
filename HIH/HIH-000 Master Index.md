# HIH-000 Master Index

## Purpose

This master index coordinates the completed HermesNet Implementation Handbook. It defines the implementation philosophy, complete volume map, dependency order, and review gates for converting HES into Rust implementation work.

## Implementation philosophy

HermesNet is implemented as deterministic Rust systems software. The hot path is small, single-purpose, allocation-controlled, and free of blocking infrastructure. Cold-path services may use richer libraries, but they must never leak nondeterminism into matching, risk reservation, settlement, or replay.

## Volume map

| Volume | Title | Status |
|---|---|---|
| HIH-001 | Rust Workspace & Crate Architecture | Complete |
| HIH-002 | Gateway Implementation | Complete |
| HIH-003 | Book Core Implementation | Complete |
| HIH-004 | Risk & Reservation Implementation | Complete |
| HIH-005 | Clearing & Event Log Implementation | Complete |
| HIH-006 | Wallet & Ledger Implementation | Complete |
| HIH-007 | Futures & Liquidation Implementation | Complete |
| HIH-008 | Market Data & Connectivity Implementation | Complete |
| HIH-009 | Testing & Benchmarking Implementation | Complete |
| HIH-010 | Deployment & Operations Implementation | Complete |

## Implementation order

1. Foundation: `hermes-fixed`, `hermes-ids`, `hermes-domain`.
2. Event substrate: `hermes-events`, event serialization, hash chain, replay compatibility.
3. Hot-path book: `hermes-book`, `hermes-matching`, book-local sequence, single-writer loop, arenas, pools, snapshots.
4. Risk and settlement path: `hermes-risk`, `hermes-clearing`, reservation lifecycle, clearing deltas, ledger deltas.
5. Replay certification: `hermes-replay`, snapshot restore, deterministic equivalence tests.
6. Gateway and connectivity ingress: `hermes-gateway`, `hermes-auth`, `hermes-rate-limit`, `hermes-connectivity`.
7. Wallet and ledger services: `hermes-wallet`, `hermes-ledger`, `hermes-treasury`, `hermes-compliance`.
8. Derivatives: `hermes-futures`, `hermes-margin`, `hermes-liquidation`, `hermes-oracle`.
9. Market data and external protocols: `hermes-market-data`, `hermes-fix`, `hermes-sbe`, drop copy.
10. Test and benchmark infrastructure: `hermes-tests`, `hermes-benches`, `hermes-fixtures`, `hermes-golden-vectors`.
11. Deployment and operations: `hermes-config`, `hermes-observability`, `hermes-admin`, manifests, scripts, runbooks.

## Dependency map

- Foundational crates (`fixed`, `ids`, `domain`) have no dependency on application crates.
- `hermes-events` depends on foundational types and is consumed by hot-path and projection crates.
- `hermes-book` owns the single-writer runtime and depends downward on matching, risk traits, clearing traits, events, fixed, and IDs.
- `hermes-risk` and `hermes-clearing` depend on domain/fixed/events and expose pure deterministic contracts to Book Core.
- `hermes-replay` depends on events plus deterministic apply contracts; runtime crates must remain replayable without DB/Kafka.
- Gateway/connectivity crates depend on domain/events and router contracts but hot-path crates do not depend on network protocols.
- Wallet/ledger/futures/market-data consume durable events and journals; they do not mutate Book Core state directly.
- Test/benchmark crates may depend broadly but must not become production dependencies.
- Deployment crates/scripts configure and supervise binaries; they must not change deterministic engine semantics.

## Crate map summary

See `crate-map/crate-map.md` for crate ownership, hot/cold classification, forbidden dependencies, and dependency direction. The completed HIH volumes provide the implementation contracts that expand that map.

## Relationship to HES volumes

HES remains the source of truth for externally observable behavior. HIH volumes translate HES requirements into Rust implementation design without adding new system obligations.

## Review gates

- Fixed-point arithmetic audit.
- ID serialization audit.
- Event schema and hash-chain audit.
- Matching determinism and single-writer audit.
- Risk reservation atomicity audit.
- Clearing, wallet, and ledger double-entry audit.
- Replay equivalence and golden-vector certification.
- Gateway security and overload audit.
- Market-data sequence/checksum audit.
- Deployment recovery and rollback audit.
