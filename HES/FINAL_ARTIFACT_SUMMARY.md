# Final Artifact Summary

## What Is Now Complete

The repository now includes practical fixtures for implementers and QA without expanding frozen HES Volumes I-X.

## Artifacts Added

- Domain examples under `examples/`.
- Golden vectors under `examples/golden-vectors/`.
- Reference EngineEvent samples under `examples/engine-events/`.
- Replay fixtures under `examples/replay/`.
- Requirements catalogue in `requirements/REQUIREMENTS_INDEX.md`.
- Improved ADR migration placeholders in `ADR/`.

## How Developers Should Use Examples

Use each domain example as a request, response, and EngineEvent contract seed when building gateway, matching, wallet, ledger, market-data, risk, liquidation, and derivatives services.

## How QA Should Use Golden Vectors

Load each golden vector as an executable conformance fixture and assert emitted events, balances, positions, order-book state, ledger entries, hash-chain results, and invariant lists.

## How Replay Samples Should Be Used

Apply `examples/replay/event-stream.json` to `snapshot-before.json`, compare the reconstructed state to `snapshot-after.json`, and verify `replay-result.json`.

## What Remains for HIH and HAT Repositories

HIH and HAT should convert these specification fixtures into code-level schemas, integration tests, replay harnesses, and CI validation jobs.
