# Crate Map

## Proposed crates

- `hermes-domain`
- `hermes-fixed`
- `hermes-ids`
- `hermes-events`
- `hermes-book`
- `hermes-matching`
- `hermes-risk`
- `hermes-clearing`
- `hermes-wallet`
- `hermes-ledger`
- `hermes-futures`
- `hermes-market-data`
- `hermes-gateway`
- `hermes-connectivity`
- `hermes-replay`
- `hermes-observability`
- `hermes-config`
- `hermes-tests`

## Dependency direction

Dependencies flow from boundary/orchestration crates toward foundational crates. Foundational crates never depend upward.

## Forbidden dependencies

Hot-path crates must not depend on databases, Kafka, HTTP clients, async runtimes, floating-point decimal libraries, global mutable state frameworks, or unbounded queue libraries.

## Hot path vs cold path

Hot path: fixed, ids, domain, events, book, matching, risk, clearing, replay core. Cold path: gateway, connectivity, market data projection, observability, config, deployment binaries.

## Ownership rules

Each crate owns its invariants and tests. Cross-crate orchestration belongs in gateway or runtime wiring, not foundational crates.
