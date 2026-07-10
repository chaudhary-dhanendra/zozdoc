# HermesNet Implementation Handbook (HIH)

HIH is the engineering handbook for implementing HermesNet in Rust. It converts the normative requirements in HES into crate boundaries, module contracts, trait contracts, build strategy, test strategy, benchmark strategy, and an implementation sequence.

## Relationship to HES

HES answers **what HermesNet must do**. HIH answers **how engineers should implement it**. HES remains authoritative. If HIH conflicts with HES, follow HES and patch HIH to remove the conflict.

HIH must preserve HES principles:

- Rust on the hot path.
- Fixed-point integers only in matching, risk, settlement, ledger, and replay.
- Single-writer Book Core per book.
- Book-local sequence numbers.
- Deterministic replay from immutable EngineEvents.
- Hash-chained append-only event log.
- No database, Kafka, network, or blocking I/O in the hot path.
- No heap allocation during steady-state matching.
- Bounded queues and explicit backpressure.
- Event append before externally visible success.

## What belongs in HIH

- Workspace and crate layout.
- Public module responsibilities.
- Trait, struct, enum, and error contracts.
- Allowed and forbidden dependencies.
- File layout for future Rust crates.
- Build, CI, testing, replay, fuzzing, and benchmark guidance.
- Implementation ordering and review gates.
- Codex task contracts for future implementation work.

## What does not belong in HIH

- New HES product requirements.
- Production Rust source files under `src/`.
- Protocol changes not already justified by HES.
- Operational runbooks that are unrelated to implementation.
- Database schema or Kafka-first designs for the hot path.

## How engineers should use HIH

1. Read `HIH-000 Master Index.md` for the sequence and dependency map.
2. Implement crates in the order defined by `HIH-001 Rust Workspace & Crate Architecture.md`.
3. Use `templates/` when opening implementation tasks.
4. Use `module-contracts/` as acceptance criteria for module-level code reviews.
5. Use `crate-map/crate-map.md` before adding dependencies.

## How Codex should edit HIH

- Do not expand HES to satisfy an HIH task.
- Do not create production Rust code unless the prompt explicitly asks for implementation.
- Prefer adding or refining contracts, examples, and checklists.
- Keep crate ownership and dependency direction explicit.
- When conflicts are discovered, state that HES is authoritative.

## Implementation sequence

1. Fixed-point types.
2. ID types.
3. Shared domain model.
4. Engine events.
5. Book storage and Book Core.
6. Matching.
7. Risk reservation.
8. Clearing and replay.
9. Gateway ingress/egress.
10. Wallet and ledger.
11. Futures and liquidation.
12. Market data and connectivity.
13. Testing, benchmarking, deployment, and operations.

## Repository layout

```text
HIH/
  README.md
  HIH-000 Master Index.md
  HIH-001 Rust Workspace & Crate Architecture.md
  HIH-002 Gateway Implementation.md
  ...
  templates/
  examples/
  crate-map/
  module-contracts/
  IMPLEMENTATION_SUMMARY.md
```
