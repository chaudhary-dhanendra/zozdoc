# HermesNet Versioning Policy

## Semantic Versions

HES releases use `MAJOR.MINOR.PATCH`.

## Major

A major version changes normative architecture, compatibility, financial behavior, replay behavior, external protocol contracts, or implementation obligations.

## Minor

A minor version adds compatible requirements, clarifications, diagrams, RFC outcomes, ADRs, examples, or optional implementation guidance.

## Patch

A patch version corrects editorial issues, broken references, formatting, non-normative examples, or approved errata that do not alter required behavior.

## ADR Revision

ADR revisions fix metadata or clarify wording. A decision change requires a new ADR that supersedes the old one.

## Volume Revision

A volume revision records approved changes to a HES volume. Frozen volumes require amendment approval before revision.

## Document Revision

Governance and support documents may revise independently, but release notes must record material policy changes.

## Release Candidate

A release candidate is a frozen candidate baseline awaiting certification, audit, and approval gates.

## Stable

Stable releases are approved, tagged, published, and treated as implementation baselines.

## Deprecated

Deprecated content remains valid for historical compatibility but is no longer recommended for new implementation.

## Obsolete

Obsolete content must not be used for new implementation and is retained only for traceability.

## Historical Archive

Historical archives preserve released baselines, superseded ADRs, rejected RFCs, and certification evidence.

## Compatibility Rules

- Patch releases must be behavior-compatible.
- Minor releases must preserve replay and financial compatibility unless explicitly scoped as optional.
- Major releases may introduce incompatible behavior only with migration guidance.
- Stable references must not be rewritten; supersede instead.
