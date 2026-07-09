# HermesNet Engineering Governance Manual

## Repository Governance

HES is governed as an engineering control repository. Markdown source is authoritative; generated artifacts are derived. Governance changes must be reviewed with the same discipline as technical specification changes because they control how implementation obligations are created, approved, frozen, and released.

## Document Ownership

Every governed document has an owner responsible for correctness, lifecycle state, review routing, and release readiness. Owners may delegate review but remain accountable for the document baseline.

## Chief Architect Responsibilities

The Chief Architect owns architectural coherence, deterministic execution principles, cross-volume consistency, release readiness, and final escalation for unresolved technical disputes.

## Technical Review Board

The Technical Review Board (TRB) reviews normative technical changes, algorithmic implications, cross-volume dependencies, replay behavior, performance commitments, and security-sensitive changes.

## ADR Approval

ADRs require owner sponsorship, TRB review, and Chief Architect acceptance when they affect architecture, determinism, public interfaces, storage, replay, or financial correctness.

## RFC Approval

RFCs move from draft to discussion, then to accepted or rejected. Accepted RFCs require an implementation plan, affected documents, compatibility assessment, and explicit owner assignment.

## Change Control Board

The Change Control Board (CCB) approves changes to frozen volumes, release candidates, stable baselines, compatibility policy, and publication artifacts.

## Version Freeze Process

A version freeze suspends normative change except approved release-blocking fixes. Freeze entry requires an open release candidate, changelog update, review checklist completion, and named approvers.

## Specification Freeze

A specification freeze locks technical behavior for a release. Editorial corrections may continue if they do not alter implementation obligations.

## Amendment Workflow

1. File amendment request with scope, rationale, risk, and affected references.
2. Obtain document-owner triage.
3. Complete domain review and TRB assessment.
4. Obtain CCB approval for frozen content.
5. Merge with changelog and version updates.

## Deprecation Workflow

Deprecation requires replacement guidance, compatibility impact, migration plan, effective version, and historical traceability. Deprecated content must remain discoverable until declared obsolete.

## Release Process

1. Confirm scope and target version.
2. Freeze normative changes.
3. Complete review gates.
4. Complete certification gates.
5. Update changelog, versioning notes, roadmap, and release metadata.
6. Tag the release and publish artifacts.
7. Archive evidence and release notes.

## Publication Process

Published artifacts must identify version, date, source commit, release status, and generated artifact process. Markdown remains the source of truth.

## Review Gates

- Gate 1: Document owner review.
- Gate 2: Domain technical review.
- Gate 3: TRB review for normative or cross-domain changes.
- Gate 4: CCB review for frozen or release-candidate changes.
- Gate 5: Chief Architect release approval.

## Certification Gates

Certification requires documented evidence for deterministic replay, financial correctness, security review, performance benchmarks, operational recovery, and documentation completeness where applicable.

## Audit Process

Audits verify that changes are traceable from issue, RFC, ADR, or amendment through review, approval, merge, release notes, and publication. Audit evidence must be retained with release records.

## Document Lifecycle

Documents move through `Draft`, `Review`, `Accepted`, `Frozen`, `Deprecated`, `Obsolete`, and `Historical` states. State changes require recorded approval and changelog entries when material.
