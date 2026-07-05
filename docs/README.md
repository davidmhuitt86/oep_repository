# OEP Repository — Architecture & Specification Index

This directory is the permanent engineering record for the **OEP Repository**,
the foundational layer of the Open Engineering Platform (OEP). It contains
the specification (not implementation notes) that any independent
organization can use to build a conformant implementation.

## Reading order

1. [architecture/00-repository-architecture.md](architecture/00-repository-architecture.md) — core concepts, philosophy applied, system boundaries
2. [architecture/01-repository-specification.md](architecture/01-repository-specification.md) — normative spec: repository layout, manifest, lifecycle
3. [architecture/02-object-storage-architecture.md](architecture/02-object-storage-architecture.md) — Engineering Object identity, revisions, content addressing
4. [architecture/03-commit-architecture.md](architecture/03-commit-architecture.md) — commit DAG, structure, semantics
5. [architecture/04-branch-architecture.md](architecture/04-branch-architecture.md) — branches, refs, merge model
6. [architecture/05-transaction-journal-architecture.md](architecture/05-transaction-journal-architecture.md) — write-ahead journal, crash recovery, sync
7. [architecture/06-storage-provider-abstraction.md](architecture/06-storage-provider-abstraction.md) — capability contracts, provider matrix
8. [architecture/07-package-format-proposal.md](architecture/07-package-format-proposal.md) — the `.oep` portable package format
9. [architecture/08-integrity-verification-strategy.md](architecture/08-integrity-verification-strategy.md) — hashing, Merkle structure, signatures, verification levels
10. [architecture/09-cli-specification.md](architecture/09-cli-specification.md) — Milestone 1 command-line interface

## Decision record

Significant architectural decisions are logged as ADRs in [adr/](adr/README.md).
The ADR log is itself part of the permanent engineering record — it is not
edited after acceptance; superseding decisions get new ADRs that reference
the ones they replace.

## Status

This is Phase 1 — architecture only. No implementation exists yet. Nothing
in this directory should be read as describing shipped behavior.
