# HermesNet Exchange Specification (HES)

## What HES Is

HES is the canonical Markdown source repository for the HermesNet Exchange Specification. It defines exchange architecture, deterministic trading behavior, accounting controls, derivatives behavior, connectivity, operations, security, quality engineering, and engineering governance.

The repository is maintained as an enterprise engineering documentation system. Markdown files are the source of truth; generated DOCX, PDF, portal, or rendered artifacts are derivative publication outputs.

## Repository Architecture

### Specification Volumes

- `HES-000 Master Index.md` — navigation, status, roadmap, dependency map, and governance rules.
- `HES-001 Volume I.md` — Foundation & Core Trading Infrastructure.
- `HES-002 Volume II.md` — Trading Kernel Internals.
- `HES-003 Volume III.md` — Algorithms & Deterministic Execution.
- `HES-004 Volume IV.md` — Wallet, Ledger & Financial Accounting.
- `HES-005 Volume V.md` — Futures, Margin & Liquidation.
- `HES-006 Volume VI.md` through `HES-010 Volume X.md` — planned placeholder volumes.

### Governance and Process

- `CONTRIBUTING.md` — contribution workflow, branch strategy, review requirements, editing rules, and release workflow.
- `GOVERNANCE.md` — repository governance, ownership, review boards, freeze process, amendment workflow, publication, certification, and audit process.
- `STYLE_GUIDE.md` — official documentation style, terminology, Markdown, Mermaid, Rust pseudocode, review checklist, and acceptance criteria rules.
- `DOCUMENT_TEMPLATE.md` — standard structure for future HES chapters and controlled engineering documents.
- `REVIEW_CHECKLIST.md` — master checklist for architecture, algorithms, security, performance, operations, replay, testing, and documentation review.
- `VERSIONING.md` — semantic versioning, document revision, lifecycle status, compatibility, and archive policy.
- `CHANGELOG.md` — release history using Keep-a-Changelog structure.
- `ROADMAP.md` — completion status, remaining volumes, and future milestones.
- `REPOSITORY_AUDIT.md` — summary of repository governance structure and remaining work.

### Engineering Control Directories

- `ADR/` — Architecture Decision Records and decision lifecycle documentation.
- `RFC/` — Requests for Comments and proposal lifecycle documentation.
- `requirements/` — requirement IDs, traceability, verification, and validation guidance.
- `diagrams/` — shared component, sequence, state, deployment, memory, and network diagrams.
- `examples/` — reference implementations, sample events, snapshots, journals, and golden vectors.
- `benchmarks/` — latency, replay, matching, memory, and certification benchmark guidance.
- `SPLIT_SUMMARY.md` — record of the initial split from `path.md`.

## Editing Workflow

1. Read `CONTRIBUTING.md`, `STYLE_GUIDE.md`, and `GOVERNANCE.md` before changing controlled content.
2. Create a scoped branch such as `feature/<scope>`, `fix/<scope>`, `adr/<number>-<slug>`, or `rfc/<number>-<slug>`.
3. Edit only one HES volume at a time unless a cross-volume change is explicitly approved.
4. Preserve technical content, diagrams, Rust pseudocode, tables, summaries, and normative statements.
5. Update cross-references, changelog entries, requirements, ADRs, RFCs, or roadmap entries when the change requires traceability.
6. Complete the relevant section of `REVIEW_CHECKLIST.md` before requesting approval.

## Frozen Volume Rule

Frozen volumes must not be modified except for corrections, clarifications, typo fixes, formatting repairs, or approved amendments. Volumes I–IV are frozen. Frozen-volume amendments require governance approval as defined in `GOVERNANCE.md`.

## Generated Artifact Rule

DOCX and PDF files are generated artifacts, not the source of truth. The Markdown files in this directory are authoritative.

## Governance Links

- Contribution workflow: `CONTRIBUTING.md`
- Governance manual: `GOVERNANCE.md`
- Style guide: `STYLE_GUIDE.md`
- Review checklist: `REVIEW_CHECKLIST.md`
- Versioning policy: `VERSIONING.md`
- Changelog: `CHANGELOG.md`

## Roadmap Links

- Repository roadmap: `ROADMAP.md`
- Master index: `HES-000 Master Index.md`
- Repository audit: `REPOSITORY_AUDIT.md`

## Contribution Links

- ADR process: `ADR/README.md`
- RFC process: `RFC/README.md`
- Requirements process: `requirements/README.md`
- Diagram conventions: `diagrams/README.md`
- Examples guidance: `examples/README.md`
- Benchmark guidance: `benchmarks/README.md`

## Recommended Next Task

Create ADR and RFC templates, add a requirements traceability matrix template, and add automated Markdown, link, and Mermaid validation without modifying technical volumes.
