# ADR-0014: Native `.oep` container format — supersedes ZIP-based packaging

Status: Accepted (Sprint 0.5 Final Directive) — **supersedes [ADR-0007](ADR-0007-package-format.md) and [ADR-0013](ADR-0013-container-format-reevaluation.md) in full**

## Context

ADR-0007 selected a deterministic ZIP archive as the `.oep` container.
ADR-0013 re-evaluated that choice against a custom binary format and
re-affirmed ZIP, primarily on tooling-ubiquity and standardization
grounds. Chief Systems Architect direction (Sprint 0.5 Final Directive)
overrides both: the repository architecture is now judged mature enough
that `.oep` should be OEP's own native, purpose-built container — the
engineering-domain equivalent of what a Git packfile is to Git, or what
PDF/DOCX became to documents — rather than a repackaging of a
general-purpose container.

## Problem Statement

Given that ADR-0013's own analysis found ZIP wins on tooling ubiquity and
standardization but a custom format wins on streaming ergonomics and
native integration with content-addressed, block-structured engineering
data — and given an explicit architectural directive to stop treating
"native format" as merely one option among several — how should `.oep`
v0.1 be designed so that the tooling-ubiquity cost is worth paying?

## Alternatives Considered

This ADR does not re-run the Option A/B comparison from ADR-0013 (already
done). It records the decision to accept Option B's trade-offs
deliberately, with two mitigations that make the cost tractable:

1. **Reference implementation as the tooling answer.** ADR-0013's
   strongest argument for ZIP was "every OS and language already has a
   reader." A native format cannot claim this on day one. The mitigation
   is that OEP will ship an open, permissively-licensed reference
   reader/writer library as a first-class deliverable of the format
   itself — the format specification and the reference implementation are
   treated as co-equal artifacts, not spec-then-maybe-tooling-later. This
   is the same strategy that made SQLite's file format broadly
   trustworthy despite being a single-project format: the reference
   implementation *is* the de facto standard's on-ramp.
2. **Design closely enough on proven primitives (block/header/footer
   layout, content addressing, CRC/hash-verified sections) that a second
   independent implementation is a tractable multi-week effort, not a
   multi-year one** — directly serving the "independent implementations"
   constitutional tenet ([00-repository-architecture.md](../architecture/00-repository-architecture.md) §5).

## Selected Solution

`.oep` v0.1 is a native, block-structured, content-addressed binary
container, specified in
[16-native-container-format.md](../architecture/16-native-container-format.md)
through
[20-package-maintenance.md](../architecture/20-package-maintenance.md).
ZIP, TAR, and other general-purpose containers become **interchange/export
targets only** ("future compatibility layers may import or export ZIP,
TAR, or other formats") — never the canonical form.

## Rationale

- A native format can be designed streaming-first, with a header-forward
  layout and block-level random access purpose-built for content-addressed
  engineering data — exactly the criterion ZIP structurally loses on in
  ADR-0013's own table (central-directory-at-end forces a two-pass read).
- A native format can encode domain semantics directly (Object blocks,
  Relationship blocks, Commit blocks — [17](../architecture/17-block-layout.md))
  rather than mapping them onto generic archive-entry-path conventions,
  which is closer to what a 50-year canonical format for an engineering
  domain should look like — comparable to how DICOM (medical imaging) or
  IFC (building/construction engineering) each defined their own container
  rather than wrapping ZIP.
- Long-term standardization is achieved differently than ADR-0013 assumed:
  not by inheriting an existing standard's credibility, but by OEP
  publishing its own open specification plus reference implementation —
  the same path PDF, DOCX (as OOXML), and SQLite each eventually took, all
  of which started as a single organization's format before becoming
  broadly standardized.

## Consequences

- [07-package-format-proposal.md](../architecture/07-package-format-proposal.md)
  is now superseded by the 16–20 series and marked accordingly; it remains
  in the document set as historical record of the rejected path, per ADR
  log conventions (ADRs and superseded specs are not deleted).
- OEP now owns a binary format specification as a permanent maintenance
  responsibility — a materially larger commitment than "reuse ZIP,"
  accepted deliberately per this directive.
- Every claim ADR-0013 made in ZIP's favor (tooling, standardization) must
  be actively re-earned over time through the reference implementation
  and open specification, not assumed for free the way adopting ZIP would
  have provided it. This is tracked as an ongoing program commitment, not
  a one-time cost paid at v0.1.

## Future Considerations

If, after a real reference implementation and at least one independent
second implementation exist, native `.oep` still proves harder to tool
around than anticipated, that would be evidence for a future ADR — not a
reason to revisit this decision speculatively now.
