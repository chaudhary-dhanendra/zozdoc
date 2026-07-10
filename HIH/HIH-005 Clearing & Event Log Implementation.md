# Clearing & Event Log Implementation

## Purpose

Define implementation guidance for clearing & event log implementation while preserving HES as the authoritative behavioral specification.

## Related HES volumes

- TODO: Link exact HES volumes during expansion.
- HES principles for determinism, fixed-point arithmetic, bounded queues, immutable events, and event append before success apply.

## Planned chapters

- EngineEvent append
- Hash chain
- Trade clearing
- Replay boundary

## Target crates

- `hermes-clearing`
- `hermes-events`
- `hermes-replay`

## Key traits

- TODO: Define public trait contracts during expansion.

## Key structs

- TODO: Define struct contracts during expansion.

## Implementation risks

- Accidental nondeterminism.
- Unbounded memory or queue growth.
- Leaking cold-path dependencies into hot-path crates.
- Event ordering mistakes.

## Testing responsibilities

- Unit tests for local invariants.
- Property tests for deterministic state transitions.
- Replay or integration tests where externally visible behavior is affected.

## TODO expansion markers

- TODO(HIH): Expand chapters.
- TODO(HIH): Add crate-local file layout.
- TODO(HIH): Add review gate checklist.
