---
document_id: HES-METADATA-STANDARD
title: HermesNet Metadata Standard
version: 0.1.0
status: Draft
owner: TODO
reviewers: [TODO]
approval_authority: TODO
created: 2026-07-10
last_updated: 2026-07-10
supersedes:
superseded_by:
related_adrs: []
related_rfcs: []
classification: Internal
---

# Metadata Standard

## Purpose

This standard defines the required YAML metadata header for all future HES documents, templates, ADRs, RFCs, reference guides, and controlled appendices.

## Required Header

```yaml
---
document_id:
title:
version:
status:
owner:
reviewers:
approval_authority:
created:
last_updated:
supersedes:
superseded_by:
related_adrs:
related_rfcs:
classification:
---
```

## Allowed Status Values

- Draft
- Review
- Approved
- Frozen
- Deprecated
- Superseded
- Archived

## Field Rules

- `document_id` must be stable, unique, and never reused for a different document.
- `version` should follow repository versioning policy.
- `owner` and `approval_authority` must name accountable roles or teams.
- `created` and `last_updated` must use ISO 8601 calendar dates.
- `related_adrs` and `related_rfcs` should use repository-relative links when known.
