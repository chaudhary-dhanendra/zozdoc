---
document_id: HES-ARCH-PRINCIPLES
title: HermesNet Architecture Principles
version: 0.1.0
status: Draft
owner: TODO
reviewers: [TODO]
approval_authority: TODO
created: TODO-YYYY-MM-DD
last_updated: TODO-YYYY-MM-DD
related_adrs: [TODO]
related_rfcs: [TODO]
classification: Internal
---
# Architecture Principles

## 1. No Global Sequencer

**Statement:** No Global Sequencer is a governing constraint for HermesNet design and implementation.

**Rationale:** The exchange must preserve deterministic decisions, bounded latency, and auditable recovery under production failure conditions.

**Implication:** Designs must document how they uphold this principle before implementation or review approval.

**Forbidden patterns:** Unbounded coordination, implicit side effects, non-deterministic hot-path dependencies, and undocumented operator bypasses.

**Related volumes:** TODO: Link applicable HES volumes.

**Related ADR placeholder:** TODO: Link the ADR that records this decision.

## 2. Book-Local Ordering

**Statement:** Book-Local Ordering is a governing constraint for HermesNet design and implementation.

**Rationale:** The exchange must preserve deterministic decisions, bounded latency, and auditable recovery under production failure conditions.

**Implication:** Designs must document how they uphold this principle before implementation or review approval.

**Forbidden patterns:** Unbounded coordination, implicit side effects, non-deterministic hot-path dependencies, and undocumented operator bypasses.

**Related volumes:** TODO: Link applicable HES volumes.

**Related ADR placeholder:** TODO: Link the ADR that records this decision.

## 3. Single-Writer Book Core

**Statement:** Single-Writer Book Core is a governing constraint for HermesNet design and implementation.

**Rationale:** The exchange must preserve deterministic decisions, bounded latency, and auditable recovery under production failure conditions.

**Implication:** Designs must document how they uphold this principle before implementation or review approval.

**Forbidden patterns:** Unbounded coordination, implicit side effects, non-deterministic hot-path dependencies, and undocumented operator bypasses.

**Related volumes:** TODO: Link applicable HES volumes.

**Related ADR placeholder:** TODO: Link the ADR that records this decision.

## 4. Fixed-Point Arithmetic

**Statement:** Fixed-Point Arithmetic is a governing constraint for HermesNet design and implementation.

**Rationale:** The exchange must preserve deterministic decisions, bounded latency, and auditable recovery under production failure conditions.

**Implication:** Designs must document how they uphold this principle before implementation or review approval.

**Forbidden patterns:** Unbounded coordination, implicit side effects, non-deterministic hot-path dependencies, and undocumented operator bypasses.

**Related volumes:** TODO: Link applicable HES volumes.

**Related ADR placeholder:** TODO: Link the ADR that records this decision.

## 5. Event Sourcing

**Statement:** Event Sourcing is a governing constraint for HermesNet design and implementation.

**Rationale:** The exchange must preserve deterministic decisions, bounded latency, and auditable recovery under production failure conditions.

**Implication:** Designs must document how they uphold this principle before implementation or review approval.

**Forbidden patterns:** Unbounded coordination, implicit side effects, non-deterministic hot-path dependencies, and undocumented operator bypasses.

**Related volumes:** TODO: Link applicable HES volumes.

**Related ADR placeholder:** TODO: Link the ADR that records this decision.

## 6. Deterministic Replay

**Statement:** Deterministic Replay is a governing constraint for HermesNet design and implementation.

**Rationale:** The exchange must preserve deterministic decisions, bounded latency, and auditable recovery under production failure conditions.

**Implication:** Designs must document how they uphold this principle before implementation or review approval.

**Forbidden patterns:** Unbounded coordination, implicit side effects, non-deterministic hot-path dependencies, and undocumented operator bypasses.

**Related volumes:** TODO: Link applicable HES volumes.

**Related ADR placeholder:** TODO: Link the ADR that records this decision.

## 7. Hot/Cold Path Separation

**Statement:** Hot/Cold Path Separation is a governing constraint for HermesNet design and implementation.

**Rationale:** The exchange must preserve deterministic decisions, bounded latency, and auditable recovery under production failure conditions.

**Implication:** Designs must document how they uphold this principle before implementation or review approval.

**Forbidden patterns:** Unbounded coordination, implicit side effects, non-deterministic hot-path dependencies, and undocumented operator bypasses.

**Related volumes:** TODO: Link applicable HES volumes.

**Related ADR placeholder:** TODO: Link the ADR that records this decision.

## 8. No Database in Hot Path

**Statement:** No Database in Hot Path is a governing constraint for HermesNet design and implementation.

**Rationale:** The exchange must preserve deterministic decisions, bounded latency, and auditable recovery under production failure conditions.

**Implication:** Designs must document how they uphold this principle before implementation or review approval.

**Forbidden patterns:** Unbounded coordination, implicit side effects, non-deterministic hot-path dependencies, and undocumented operator bypasses.

**Related volumes:** TODO: Link applicable HES volumes.

**Related ADR placeholder:** TODO: Link the ADR that records this decision.

## 9. No Kafka Before Decision

**Statement:** No Kafka Before Decision is a governing constraint for HermesNet design and implementation.

**Rationale:** The exchange must preserve deterministic decisions, bounded latency, and auditable recovery under production failure conditions.

**Implication:** Designs must document how they uphold this principle before implementation or review approval.

**Forbidden patterns:** Unbounded coordination, implicit side effects, non-deterministic hot-path dependencies, and undocumented operator bypasses.

**Related volumes:** TODO: Link applicable HES volumes.

**Related ADR placeholder:** TODO: Link the ADR that records this decision.

## 10. Immutable EngineEvents

**Statement:** Immutable EngineEvents is a governing constraint for HermesNet design and implementation.

**Rationale:** The exchange must preserve deterministic decisions, bounded latency, and auditable recovery under production failure conditions.

**Implication:** Designs must document how they uphold this principle before implementation or review approval.

**Forbidden patterns:** Unbounded coordination, implicit side effects, non-deterministic hot-path dependencies, and undocumented operator bypasses.

**Related volumes:** TODO: Link applicable HES volumes.

**Related ADR placeholder:** TODO: Link the ADR that records this decision.

## 11. Hash-Chained Event Log

**Statement:** Hash-Chained Event Log is a governing constraint for HermesNet design and implementation.

**Rationale:** The exchange must preserve deterministic decisions, bounded latency, and auditable recovery under production failure conditions.

**Implication:** Designs must document how they uphold this principle before implementation or review approval.

**Forbidden patterns:** Unbounded coordination, implicit side effects, non-deterministic hot-path dependencies, and undocumented operator bypasses.

**Related volumes:** TODO: Link applicable HES volumes.

**Related ADR placeholder:** TODO: Link the ADR that records this decision.

## 12. Idempotency

**Statement:** Idempotency is a governing constraint for HermesNet design and implementation.

**Rationale:** The exchange must preserve deterministic decisions, bounded latency, and auditable recovery under production failure conditions.

**Implication:** Designs must document how they uphold this principle before implementation or review approval.

**Forbidden patterns:** Unbounded coordination, implicit side effects, non-deterministic hot-path dependencies, and undocumented operator bypasses.

**Related volumes:** TODO: Link applicable HES volumes.

**Related ADR placeholder:** TODO: Link the ADR that records this decision.

## 13. Bounded Memory

**Statement:** Bounded Memory is a governing constraint for HermesNet design and implementation.

**Rationale:** The exchange must preserve deterministic decisions, bounded latency, and auditable recovery under production failure conditions.

**Implication:** Designs must document how they uphold this principle before implementation or review approval.

**Forbidden patterns:** Unbounded coordination, implicit side effects, non-deterministic hot-path dependencies, and undocumented operator bypasses.

**Related volumes:** TODO: Link applicable HES volumes.

**Related ADR placeholder:** TODO: Link the ADR that records this decision.

## 14. Bounded Queues

**Statement:** Bounded Queues is a governing constraint for HermesNet design and implementation.

**Rationale:** The exchange must preserve deterministic decisions, bounded latency, and auditable recovery under production failure conditions.

**Implication:** Designs must document how they uphold this principle before implementation or review approval.

**Forbidden patterns:** Unbounded coordination, implicit side effects, non-deterministic hot-path dependencies, and undocumented operator bypasses.

**Related volumes:** TODO: Link applicable HES volumes.

**Related ADR placeholder:** TODO: Link the ADR that records this decision.

## 15. Backpressure over Collapse

**Statement:** Backpressure over Collapse is a governing constraint for HermesNet design and implementation.

**Rationale:** The exchange must preserve deterministic decisions, bounded latency, and auditable recovery under production failure conditions.

**Implication:** Designs must document how they uphold this principle before implementation or review approval.

**Forbidden patterns:** Unbounded coordination, implicit side effects, non-deterministic hot-path dependencies, and undocumented operator bypasses.

**Related volumes:** TODO: Link applicable HES volumes.

**Related ADR placeholder:** TODO: Link the ADR that records this decision.

## 16. Fail Fast

**Statement:** Fail Fast is a governing constraint for HermesNet design and implementation.

**Rationale:** The exchange must preserve deterministic decisions, bounded latency, and auditable recovery under production failure conditions.

**Implication:** Designs must document how they uphold this principle before implementation or review approval.

**Forbidden patterns:** Unbounded coordination, implicit side effects, non-deterministic hot-path dependencies, and undocumented operator bypasses.

**Related volumes:** TODO: Link applicable HES volumes.

**Related ADR placeholder:** TODO: Link the ADR that records this decision.

## 17. Auditability

**Statement:** Auditability is a governing constraint for HermesNet design and implementation.

**Rationale:** The exchange must preserve deterministic decisions, bounded latency, and auditable recovery under production failure conditions.

**Implication:** Designs must document how they uphold this principle before implementation or review approval.

**Forbidden patterns:** Unbounded coordination, implicit side effects, non-deterministic hot-path dependencies, and undocumented operator bypasses.

**Related volumes:** TODO: Link applicable HES volumes.

**Related ADR placeholder:** TODO: Link the ADR that records this decision.

## 18. Security by Design

**Statement:** Security by Design is a governing constraint for HermesNet design and implementation.

**Rationale:** The exchange must preserve deterministic decisions, bounded latency, and auditable recovery under production failure conditions.

**Implication:** Designs must document how they uphold this principle before implementation or review approval.

**Forbidden patterns:** Unbounded coordination, implicit side effects, non-deterministic hot-path dependencies, and undocumented operator bypasses.

**Related volumes:** TODO: Link applicable HES volumes.

**Related ADR placeholder:** TODO: Link the ADR that records this decision.

## 19. Operational Recoverability

**Statement:** Operational Recoverability is a governing constraint for HermesNet design and implementation.

**Rationale:** The exchange must preserve deterministic decisions, bounded latency, and auditable recovery under production failure conditions.

**Implication:** Designs must document how they uphold this principle before implementation or review approval.

**Forbidden patterns:** Unbounded coordination, implicit side effects, non-deterministic hot-path dependencies, and undocumented operator bypasses.

**Related volumes:** TODO: Link applicable HES volumes.

**Related ADR placeholder:** TODO: Link the ADR that records this decision.
