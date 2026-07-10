# HIH-000 Master Index

## Purpose

This master index coordinates the HermesNet Implementation Handbook. It defines the implementation philosophy, the volume map, the dependency order, and the review gates for converting HES into a Rust implementation.

## Implementation philosophy

HermesNet is implemented as deterministic Rust systems software. The hot path is small, single-purpose, allocation-controlled, and free of blocking infrastructure. Cold-path services may use richer libraries, but they must never leak nondeterminism into matching, risk reservation, settlement, or replay.

## Volume map

| Volume | Title | Status |
|---|---|---|
| HIH-001 | Rust Workspace & Crate Architecture | Expanded |
| HIH-002 | Gateway Implementation | Outline |
| HIH-003 | Book Core Implementation | Outline |
| HIH-004 | Risk & Reservation Implementation | Outline |
| HIH-005 | Clearing & Event Log Implementation | Outline |
| HIH-006 | Wallet & Ledger Implementation | Outline |
| HIH-007 | Futures & Liquidation Implementation | Outline |
| HIH-008 | Market Data & Connectivity Implementation | Outline |
| HIH-009 | Testing & Benchmarking Implementation | Outline |
| HIH-010 | Deployment & Operations Implementation | Outline |

## Implementation order

1. `hermes-fixed`
2. `hermes-ids`
3. `hermes-domain`
4. `hermes-events`
5. `hermes-book`
6. `hermes-matching`
7. `hermes-risk`
8. `hermes-clearing`
9. `hermes-replay`
10. `hermes-gateway`
11. `hermes-wallet` and `hermes-ledger`
12. `hermes-futures`
13. `hermes-market-data` and `hermes-connectivity`
14. `hermes-tests`, benchmarks, deployment packaging

## Dependency map

Foundational crates (`fixed`, `ids`, `domain`, `events`) must not depend on application crates. Hot-path crates depend only downward. Cold-path adapters depend on hot-path APIs, never the reverse.

## Crate map summary

See `crate-map/crate-map.md` for crate ownership, hot/cold classification, forbidden dependencies, and dependency direction.

## Relationship to HES volumes

HES remains the source of truth for externally observable behavior. HIH volumes translate HES requirements into Rust implementation design without adding new system obligations.

## Rust toolchain assumptions

- Stable Rust by default.
- MSRV is set intentionally when the workspace is created.
- `cargo fmt`, `cargo clippy`, `cargo test`, property tests, replay tests, and criterion-style benchmarks are expected.
- Nightly-only features require explicit review and must not be required for production builds.

## Build philosophy

Builds should be reproducible, feature-gated, and minimal. Release hot-path builds should enable deterministic optimizations and forbid accidental debug assertions that alter timing-sensitive behavior.

## Testing philosophy

Use layered testing: unit tests for arithmetic and IDs, property tests for matching and replay, integration tests for gateway-to-event flow, and differential replay tests for event log recovery.

## Benchmark philosophy

Benchmarks must measure book-local throughput, tail latency, allocation counts, queue pressure, and replay speed. Benchmarks are not correctness substitutes.

## Review gates

- Fixed-point arithmetic audit.
- ID serialization audit.
- Event schema and hash-chain audit.
- Matching determinism audit.
- Risk reservation atomicity audit.
- Replay equivalence audit.
- Hot-path allocation audit.
- Dependency audit.

## Remaining HIH work

Expand HIH-002 through HIH-010 after HIH-001 is accepted, keeping each expansion constrained to implementation guidance and HES alignment.
