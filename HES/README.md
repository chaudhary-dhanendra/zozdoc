# HermesNet Exchange Specification (HES)

## What HES Is

HES is the canonical Markdown source repository for the HermesNet Exchange Specification. It splits the exchange specification into professional multi-volume documents covering foundation architecture, trading internals, deterministic algorithms, wallet and ledger accounting, derivatives, connectivity, operations, security, quality engineering, and engineering standards.

## Repository Layout

- `HES-000 Master Index.md` — navigation, status, roadmap, dependency map, and governance rules.
- `HES-001 Volume I.md` — Foundation & Core Trading Infrastructure.
- `HES-002 Volume II.md` — Trading Kernel Internals.
- `HES-003 Volume III.md` — Algorithms & Deterministic Execution.
- `HES-004 Volume IV.md` — Wallet, Ledger & Financial Accounting.
- `HES-005 Volume V.md` — Futures, Margin & Liquidation.
- `HES-006 Volume VI.md` through `HES-010 Volume X.md` — planned placeholder volumes.
- `SPLIT_SUMMARY.md` — record of the initial split from `path.md`.

## How Codex Should Edit the Repository

- Edit only one volume at a time unless a user explicitly requests a cross-volume change.
- Preserve technical content, diagrams, Rust pseudocode, tables, and summaries.
- Do not summarize, compress, or delete valid technical material.
- Keep cross-reference changes conservative and obvious.

## Frozen Volume Rule

Frozen volumes must not be modified except for corrections, clarifications, typo fixes, formatting repairs, or approved amendments. Volumes I–IV are frozen.

## Generated Artifact Rule

DOCX and PDF files are generated artifacts, not the source of truth. The Markdown files in this directory are authoritative.

## Recommended Next Task

Continue expanding `HES-005 Volume V.md`, which is currently in progress.
