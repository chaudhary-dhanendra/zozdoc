# HermesNet Master Engineering Review Checklist

## Architecture

- [ ] Boundaries, ownership, and dependencies are explicit.
- [ ] Invariants are stated and testable.
- [ ] Changes align with approved ADRs and RFCs.

## Algorithms

- [ ] Deterministic ordering is preserved.
- [ ] Preconditions, postconditions, and error paths are documented.
- [ ] Complexity and edge cases are reviewed.

## Security

- [ ] Trust boundaries are identified.
- [ ] Authentication, authorization, and audit requirements are clear.
- [ ] Abuse cases and data-integrity risks are addressed.

## Performance

- [ ] Latency, throughput, and capacity implications are documented.
- [ ] Benchmark expectations are measurable.
- [ ] Hot paths avoid unnecessary allocation and nondeterminism.

## Concurrency

- [ ] Shared-state access is specified.
- [ ] Ordering, races, and deadlock risks are reviewed.
- [ ] Recovery under concurrent failure is defined.

## Memory

- [ ] Memory ownership and lifecycle are clear.
- [ ] Bounded and unbounded structures are identified.
- [ ] Snapshot and replay memory behavior is reviewed.

## Operations

- [ ] Deployment, rollback, and recovery procedures are defined.
- [ ] Operator actions are auditable.
- [ ] Runbook impact is identified.

## Observability

- [ ] Logs, metrics, traces, and audit events are specified.
- [ ] Failure signals are actionable.
- [ ] Sensitive data is not exposed.

## Financial Correctness

- [ ] Ledger, wallet, margin, and settlement effects are balanced.
- [ ] Rounding, precision, and overflow behavior are specified.
- [ ] Reconciliation and audit trails are preserved.

## Replay

- [ ] Replay inputs and ordering are authoritative.
- [ ] Output equivalence is verifiable.
- [ ] Snapshot compatibility is documented.

## Testing

- [ ] Unit, integration, property, replay, and conformance tests are identified.
- [ ] Golden vectors are updated when applicable.
- [ ] Acceptance criteria are measurable.

## Documentation

- [ ] Style guide is followed.
- [ ] Cross-references are stable.
- [ ] Diagrams, tables, and pseudocode match normative text.
