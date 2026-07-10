# Module Contract: clearing

## Responsibility

Define and enforce the `clearing` implementation boundary.

## Inputs

- Validated domain values from lower or boundary layers.

## Outputs

- Typed results, immutable events, or deterministic state transitions.

## Public traits

- TODO: Add exact traits during implementation expansion.

## Core structs

- TODO: Add exact structs during implementation expansion.

## Forbidden behavior

- Floating-point arithmetic in critical accounting or matching paths.
- Blocking I/O in hot-path modules.
- Unbounded allocation or queues.
- Wall-clock reads during deterministic processing.
- External success before required event append.

## Tests required

- Unit invariant tests.
- Property tests for state transitions.
- Replay tests where events are emitted.

## HES references

- TODO: Link exact HES sections during expansion.
