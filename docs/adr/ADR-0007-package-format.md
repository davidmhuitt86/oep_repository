# ADR-0007: `.oep` is a deterministic content-addressed archive, not a SQLite file

Status: Accepted

## Context

`project.oep` must be a single portable file a user can move, send, or
open on any conformant implementation, containing a complete Repository
snapshot (07-package-format-proposal.md).

## Problem Statement

What container format should back `.oep` — a single-file embedded
database (SQLite), a raw directory tree, or a deterministic archive of
content-addressed entries?

## Alternatives Considered

1. **SQLite file.** Excellent single-file portability and free ACID
   transactions, but binary format is not meaningfully diffable, requires
   every implementation to embed or link a SQLite engine specifically for
   *packaging* (not just as one possible Storage Provider), and couples
   the format's future to one engine's file format evolution.
2. **Raw directory tree.** Best transparency and diffability, but fails
   the "user sees one file" requirement outright.
3. **Deterministic content-addressed archive (ZIP container, canonical
   ordering, fixed timestamps, STORE method for content entries).**

## Selected Solution

Deterministic content-addressed ZIP archive, specified as OEP Package
Format v1 (07-package-format-proposal.md §2-3).

## Rationale

- ZIP readers/writers are available in essentially every language's
  standard library, so requiring one is a far smaller, more stable
  dependency than requiring a SQLite implementation specifically for
  packaging.
- Canonical entry ordering and fixed timestamps make packaging
  deterministic — a requirement extended from the Repository's own
  conformance guarantee (01-repository-specification.md §5).
- STORE (uncompressed) for content entries lets a verifier check content
  hashes directly against raw archive bytes without a decompression step.
- Keeps the package format decoupled from any Storage Provider choice —
  SQLite remains an excellent Provider choice (06-storage-provider-abstraction.md
  §7) without becoming an architectural requirement for the *distribution*
  format.

## Consequences

- Diffing two `.oep` files at the byte level is still not meaningful for
  the whole file, but individual entries can be extracted and diffed
  independently — better than an opaque database file, though not as good
  as a raw tree.
- Implementations must produce canonical (sorted, fixed-timestamp) ZIP
  output to preserve determinism — a straightforward but non-default mode
  for most ZIP libraries, called out explicitly so implementers don't ship
  a non-deterministic packer by accident.

## Future Considerations

Shallow and delta package variants (07-package-format-proposal.md §5) can
be added as new `package_kind` values within the same container format
without revisiting this decision.
