# HermesNet Documentation Style Guide

## Heading Hierarchy

Use one `#` title per document. Use `##` for primary sections, `###` for subsections, and `####` only for detailed checklists or examples. Do not skip heading levels.

## Section Numbering

Volumes may use stable section numbers when they support cross-reference traceability. Governance documents may omit numbering unless they define lifecycle stages, gates, or controlled procedures.

## Writing Tone

Use precise engineering language. Prefer active voice, observable behavior, and explicit responsibilities. Avoid marketing language, vague adjectives, and unstated assumptions.

## Terminology Rules

Use one canonical term for one concept. Define domain terms before relying on them. Do not introduce synonyms for matching, replay, ledger, margin, liquidation, wallet, snapshot, or journal semantics without an ADR.

## Engineering Vocabulary

- Use “deterministic” only when the same input stream produces the same output stream.
- Use “authoritative” for source-of-truth systems or documents.
- Use “idempotent” only when repeated application has no additional effect.
- Use “atomic” only when partial effects are impossible by contract.

## RFC Terminology

RFCs propose changes, collect discussion, and record acceptance or rejection. Use `MUST`, `SHOULD`, and `MAY` only when the RFC intends normative behavior.

## Requirement Terminology

Use RFC 2119-style terms consistently:

- `MUST` for mandatory requirements.
- `MUST NOT` for prohibited behavior.
- `SHOULD` for recommended behavior with documented exceptions.
- `MAY` for optional behavior.

## ADR Terminology

ADRs record decisions. Use “Context,” “Decision,” “Consequences,” and “Status.” Avoid reopening accepted ADRs; supersede them with a new ADR.

## Mermaid Conventions

Use stable labels, deterministic ordering, and concise edges. Keep diagrams reviewable in Markdown. Prefer `flowchart LR`, `sequenceDiagram`, and `stateDiagram-v2`.

## Rust Pseudocode Conventions

Pseudocode should be implementation-shaped without pretending to be a complete crate. Use explicit inputs, outputs, error paths, deterministic collections, and checked arithmetic.

## Tables

Tables must have clear headers, stable row order, and concise cells. Use tables for state values, approvals, ownership, compatibility, and acceptance criteria.

## Code Blocks

Always specify a language: `rust`, `mermaid`, `json`, `toml`, `text`, or `markdown`. Use `text` for logs and non-code examples.

## Warning Blocks

Use warning blocks for constraints that can cause incorrect implementation or operational risk.

> **Warning**
> A warning identifies a condition that may break determinism, financial correctness, safety, or recoverability.

## Note Blocks

Use notes for clarifications that do not change requirements.

> **Note**
> A note provides context without creating a normative requirement.

## Review Checklist Format

Use checkboxes grouped by domain:

```markdown
### Review Checklist

- [ ] Architecture impact reviewed.
- [ ] Determinism preserved.
- [ ] Cross-references updated.
```

## Acceptance Criteria Format

Acceptance criteria must be verifiable:

```markdown
### Acceptance Criteria

- [ ] Given the same journal and snapshot, replay produces identical state.
- [ ] All normative requirements have traceability identifiers.
```

## Example Formatting

Examples must be labeled as non-normative unless explicitly part of a requirement. Keep examples short and isolate assumptions.

## Naming Conventions

- ADR files: `ADR-0001-short-title.md`.
- RFC files: `RFC-0001-short-title.md`.
- Requirements: `HES-REQ-<DOMAIN>-0001`.
- Diagrams: `<domain>-<view>-<version>.mmd` when stored as files.

## Document Metadata

Governed documents should include owner, status, version, last reviewed date, and approval authority.

## Cross-Volume References

Reference volumes by exact title and stable section title. When available, include requirement, ADR, RFC, or diagram identifiers.
