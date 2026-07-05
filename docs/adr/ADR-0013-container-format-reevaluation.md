# ADR-0013: Container format re-evaluation — deterministic ZIP retained over a custom OEP container

Status: Accepted (RC2) — re-affirms [ADR-0007](ADR-0007-package-format.md) after formal re-evaluation; does not change the RC1 decision

## Context

ADR-0007 selected a deterministic, content-addressed ZIP archive as the
`.oep` package container over a SQLite file and a raw directory tree.
Sprint 0 review directed a fresh, formal comparison against a purpose-built
custom OEP container format, evaluated against a wider criteria set than
ADR-0007 used, rather than assuming ZIP is settled.

## Problem Statement

Given the full set of long-term criteria (maintainability, streaming,
performance, compression, signing, random access, tooling, cross-platform
support, forward compatibility, complexity, standardization), is a
deterministic ZIP container still the right choice, or does a custom OEP
container format now win?

## Alternatives Considered

**Option A — Deterministic ZIP container** (canonical entry ordering,
fixed timestamps, STORE method for content-addressed entries, as specified
in 07-package-format-proposal.md).

**Option B — Custom OEP container format**: a purpose-built binary framing
designed specifically around content-addressed engineering data (e.g. a
fixed header, a central directory of `{hash, offset, length, codec}`
entries, native support for per-entry compression codec selection and
per-entry streaming without ZIP's central-directory-at-end convention).

| Criterion | Option A: Deterministic ZIP | Option B: Custom OEP container | Winner |
|---|---|---|---|
| **Long-term maintainability** | ZIP's format has been stable and documented since 1989; no OEP-specific parser to maintain across decades. | Requires OEP to own, document, and maintain a binary format spec indefinitely, including handling its own versioning/evolution. | **A** |
| **Streaming** | ZIP's central directory is conventionally at the end of the file, which historically hurt pure streaming reads; workable with a "read central directory first" two-pass approach or ZIP64 streaming extensions, but not ZIP's most natural mode. | Can be designed streaming-first from scratch (e.g. header-first framing with inline entry metadata). | **B** |
| **Performance** | Mature, highly optimized implementations exist in every ecosystem; STORE method avoids decompression overhead for content entries. | Could theoretically be tuned further for OEP's specific access patterns, but starts with zero implementation maturity. | **A** (in practice, given implementation maturity outweighs theoretical headroom) |
| **Compression** | Per-entry codec choice already supported (STORE for content, DEFLATE for manifest/index); can add Zstd via ZIP's method-ID extension mechanism without breaking the container. | Could pick a single modern codec (e.g. Zstd) as the only option, slightly simpler but less flexible per-entry. | **A** (extensibility without a new container spec) |
| **Signing** | Detached signatures over canonical archive content work today (07-package-format-proposal.md §6); no in-format signature slot. | Could reserve a native signature block in the header. | **B** (marginal — detached signing already fully satisfies the requirement, so this is a minor convenience, not a capability gap) |
| **Random access** | Excellent — ZIP's central directory gives O(1) entry lookup by name once read. | Can be designed for the same or better. | **Tie** |
| **Tooling** | Every OS, every language, and countless GUI tools can open/inspect a `.oep` file's contents without any OEP-specific tooling — directly serves "users see one project" transparency. | Requires bespoke tooling (even a hex-dump-level inspection needs the OEP spec) for any inspection, which actively works against approachability and debuggability for a 20-year-old file someone finds later. | **A**, decisively |
| **Cross-platform support** | Universal. | Would need cross-platform tooling built and maintained by the OEP project itself. | **A** |
| **Forward compatibility** | New entry types are just new archive paths; old readers ignore unrecognized ones gracefully as ZIP already permits. | Requires the custom format's own extension mechanism to be designed correctly up front — a well-known hard problem (see how many bespoke binary formats get this wrong on the first attempt). | **A** |
| **Complexity** | Low — reuse an existing, battle-tested format and library ecosystem. | High — designing, specifying, and testing a new binary format to the rigor this project's 20-year standard requires is a substantial undertaking with real risk of subtle bugs (this is exactly the class of problem RFC 9562, BLAKE3, and ZIP itself all had many implementation cycles to mature past). | **A**, decisively |
| **Standardization** | ZIP is an open, ubiquitously implemented format (originally PKWARE, now de facto universal, and the base of ISO/IEC 21320-1). | Would be an OEP-specific format with zero external standardization until/unless the platform itself becomes a recognized standard — circular, since the container is supposed to help *establish* that credibility, not depend on it already existing. | **A**, decisively |

## Selected Solution

**Retain Option A — the deterministic content-addressed ZIP container —
as specified in
[07-package-format-proposal.md](../architecture/07-package-format-proposal.md).**
No change to the RC1 decision.

## Rationale

Option B wins narrowly on two criteria (pure streaming ergonomics, a
native signature slot), both of which Option A already has adequate,
already-specified answers for (a two-pass streaming read via the central
directory; detached signatures over canonical content). Option A wins
decisively on the criteria that matter most for a format meant to remain
readable, toolable, and trustworthy for 20+ years: maintainability,
tooling ubiquity, forward compatibility, complexity, and standardization.
Building a custom format to chase marginal streaming gains would trade a
large, durable advantage (existing universal tooling) for a small,
theoretical one (marginally more elegant streaming), and would add a
multi-decade maintenance burden — a new binary format is a project the
platform would own forever, distinct from and in addition to the
engineering platform itself.

## Consequences

- No specification changes required to
  [07-package-format-proposal.md](../architecture/07-package-format-proposal.md);
  this ADR exists to record that the alternative was formally
  re-evaluated against the fuller criteria set, not skipped.
- If a genuine streaming-critical use case emerges later (e.g. packaging
  multi-terabyte attachment-heavy repositories in bandwidth-constrained
  environments) that ZIP's two-pass model cannot practically serve, that
  should be scoped as a targeted extension (e.g. a streaming-optimized
  entry ordering convention within the existing ZIP container, or the
  shallow/delta package variants already noted in
  07-package-format-proposal.md §5) before a full custom-format proposal
  is considered again.

## Future Considerations

This decision should be revisited only if a concrete, measured limitation
of the ZIP-based approach is found in practice — not on a re-review
schedule, and not speculatively. Re-litigating container format without
new evidence is exactly the kind of process cost the constitutional
principles caution against paying repeatedly for the same question.
