# Contributing to the HermesNet Engineering Specification

## Purpose

The HermesNet Engineering Specification (HES) repository is the authoritative source for exchange architecture, deterministic trading behavior, financial accounting, operational controls, and implementation contracts. Contributions must improve clarity, traceability, correctness, or governance without weakening determinism or auditability.

## Contribution Workflow

1. Open or reference an issue, RFC, ADR, or approved work item.
2. Create a scoped branch from the current integration branch.
3. Edit one document area at a time and keep unrelated changes out of the branch.
4. Run Markdown, link, diagram, and review checks available to the project.
5. Open a pull request using the required checklist.
6. Address review comments with additional commits; do not force-push away review history after approval unless requested by maintainers.

## Branch Strategy

- `main` contains released or release-candidate documentation.
- `release/vX.Y` contains stabilization work for a named release.
- `feature/<scope>` is used for approved content additions.
- `fix/<scope>` is used for corrections and editorial repairs.
- `adr/<number>-<slug>` and `rfc/<number>-<slug>` are used for governance proposals.

## Review Process

Every non-trivial change requires technical review and document-owner approval. Changes affecting frozen volumes, requirements, security, accounting, replay, matching, margin, liquidation, or external behavior require governance review before merge.

## Pull Request Requirements

A pull request must include:

- Purpose and scope.
- Affected files and volumes.
- Change classification: major, minor, patch, editorial, ADR, RFC, or governance.
- Risk assessment.
- Cross-reference impact.
- Test, lint, or review checklist results.
- Required approvals.

## Editing HES

Edit Markdown as source. Preserve existing technical meaning, section identifiers, diagrams, pseudocode, tables, and normative requirements unless the change is explicitly approved.

## One-Volume-at-a-Time Rule

Contributors must edit only one HES volume per pull request unless the Technical Review Board approves a cross-volume consistency change. Repository governance files, ADRs, RFCs, and indexes may accompany a volume change only when necessary for traceability.

## Frozen Volume Policy

Frozen volumes are stable engineering baselines. They may receive typo fixes, broken-link repairs, formatting corrections, approved errata, or approved amendments. Do not rewrite frozen technical sections, algorithms, or architecture prose.

## Markdown Style Rules

- Use ATX headings (`#`, `##`, `###`).
- Keep heading hierarchy strict; do not skip levels.
- Prefer short paragraphs and explicit lists.
- Use fenced code blocks with language identifiers.
- Use tables for matrices, ownership, and state transitions.
- Keep line wrapping readable and avoid trailing whitespace.

## Mermaid Guidelines

- Use fenced `mermaid` blocks.
- Name participants and nodes with stable engineering terms.
- Prefer left-to-right flow for component diagrams.
- Use sequence diagrams for protocol interaction and state diagrams for lifecycle transitions.
- Do not encode algorithmic truth only in diagrams; diagrams must support normative text.

## Rust Pseudocode Guidelines

- Use fenced `rust` blocks for pseudocode.
- Mark non-compiling examples with comments when appropriate.
- Prefer explicit types, deterministic ordering, checked arithmetic, and error-return paths.
- Do not introduce unsafe patterns unless the specification requires and justifies them.

## Cross-Reference Policy

Cross-volume references must use stable document names, section titles, and requirement or ADR identifiers when available. Avoid fragile references such as “above,” “below,” or “later.”

## Diagram Policy

Repository-level diagrams belong under `diagrams/`. Diagrams embedded in volumes must be essential to understanding the local text. Shared diagrams require ownership, review, and versioning notes.

## Commit Message Conventions

Use imperative commits with a scope:

- `docs(governance): add contribution workflow`
- `docs(volume-v): clarify liquidation terminology`
- `adr: add deterministic replay decision`
- `fix(readme): repair roadmap link`

## Document Ownership

Each governed document must have an owner accountable for correctness, review routing, and lifecycle state. Ownership does not bypass review requirements.

## Approval Workflow

- Editorial changes: document owner.
- Technical clarifications: document owner plus domain reviewer.
- Normative behavior: Technical Review Board.
- Frozen volume amendment: Chief Architect and Change Control Board.
- Release publication: Chief Architect and release manager.

## Release Workflow

1. Cut a release branch or candidate tag.
2. Freeze normative changes.
3. Complete review and certification gates.
4. Update `CHANGELOG.md`, `VERSIONING.md` status, and roadmap state.
5. Publish signed release artifacts and archive historical baselines.
