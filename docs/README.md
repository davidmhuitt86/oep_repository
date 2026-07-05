# OEP Repository — Architecture & Specification Index

This directory is the permanent engineering record for the **OEP Repository**,
the foundational layer of the Open Engineering Platform (OEP). It contains
the specification (not implementation notes) that any independent
organization can use to build a conformant implementation.

## Reading order

1. [architecture/00-repository-architecture.md](architecture/00-repository-architecture.md) — core concepts, philosophy applied, system boundaries (constitutional five-primitive rule, RC2)
2. [architecture/01-repository-specification.md](architecture/01-repository-specification.md) — normative spec: repository layout, manifest, lifecycle
3. [architecture/02-object-storage-architecture.md](architecture/02-object-storage-architecture.md) — three-part Engineering Object identity (RC2), revisions, content addressing
4. [architecture/03-commit-architecture.md](architecture/03-commit-architecture.md) — commit DAG, structure, semantics
5. [architecture/04-branch-architecture.md](architecture/04-branch-architecture.md) — branches, refs, merge model
6. [architecture/05-transaction-journal-architecture.md](architecture/05-transaction-journal-architecture.md) — write-ahead, append-only journal, crash recovery, sync
7. [architecture/06-storage-provider-abstraction.md](architecture/06-storage-provider-abstraction.md) — capability contracts, provider matrix
8. [architecture/07-package-format-proposal.md](architecture/07-package-format-proposal.md) — the `.oep` portable package format (re-affirmed RC2)
9. [architecture/08-integrity-verification-strategy.md](architecture/08-integrity-verification-strategy.md) — hashing, Merkle structure, signatures, verification levels
10. [architecture/09-cli-specification.md](architecture/09-cli-specification.md) — Milestone 1 command-line interface
11. [architecture/10-layered-architecture.md](architecture/10-layered-architecture.md) — **(RC2)** Package / Repository / Object Store layering
12. [architecture/11-repository-internals-storage-and-encoding.md](architecture/11-repository-internals-storage-and-encoding.md) — **(Sprint 0.5)** physical layout, encoding, packing, streaming
13. [architecture/12-repository-internals-lifecycle-and-maintenance.md](architecture/12-repository-internals-lifecycle-and-maintenance.md) — **(Sprint 0.5)** GC, repair, upgrade, maintenance
14. [architecture/13-oepfs-exploration.md](architecture/13-oepfs-exploration.md) — **(exploratory, non-normative)** Open Engineering Platform File System concept

## Decision record

Significant architectural decisions are logged as ADRs in [adr/](adr/README.md).
The ADR log is itself part of the permanent engineering record — it is not
edited after acceptance; superseding decisions get new ADRs that reference
the ones they replace. RC2 added ADR-0010 through ADR-0013 and marked
ADR-0002/ADR-0007/ADR-0009 as partially superseded, elevated, or
re-affirmed respectively (see [adr/README.md](adr/README.md)).

## Status

**Release Candidate 2 (RC2).** This is still Phase 1 — architecture only.
No implementation exists yet, and none should begin against this
specification set until it is declared stable. Nothing in this directory
should be read as describing shipped behavior.

### RC1 → RC2 changes

- Object identity is now three independent parts (EOID, Revision ID,
  Content Hash) — [ADR-0010](adr/ADR-0010-three-part-identity.md).
- Package, Repository, and Object Store are now explicit, separately
  specified layers — [ADR-0011](adr/ADR-0011-three-layer-separation.md),
  [architecture/10-layered-architecture.md](architecture/10-layered-architecture.md).
- The Five-Primitive Rule is now a constitutional principle, not an
  ordinary ADR — [ADR-0012](adr/ADR-0012-five-primitive-constitutional-amendment.md).
- The `.oep` container format was formally re-evaluated against a custom
  binary format across a fuller criteria set and re-affirmed as
  deterministic ZIP — [ADR-0013](adr/ADR-0013-container-format-reevaluation.md).
- Added a Sprint 0.5 repository-internals series (storage/encoding,
  lifecycle/maintenance) and a non-normative OEPFS exploration document.
