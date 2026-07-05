# ADR-0010: Three-part Engineering Object identity (EOID / Revision ID / Content Hash)

Status: Accepted (RC2) — supersedes the revision-identity portion of [ADR-0002](ADR-0002-object-identity.md)

## Context

RC1 defined Object identity as EOID (permanent) plus a Revision ID that
was itself a hash over content and chain position
(`RevisionID = Hash(EOID || revision_index || content_hash || previous_revision_id || commit_id)`,
see ADR-0002). Sprint 0 review flagged that this makes Revision ID a
function of Content Hash, i.e. identity partially derived from content —
in tension with the constitutional separation of identity and content.

## Problem Statement

Should a revision's identity be derivable from its content (as RC1 had
it), or must Revision ID be a fully independent identity, distinct from
both the permanent EOID and the content-derived hash?

## Alternatives Considered

1. **Two-identity model (RC1):** EOID (permanent) + hash-derived Revision
   ID. Simple, but conflates "which revision is this" with "what are its
   bytes" — two revisions with identical content become indistinguishable
   as revisions without consulting chain position separately, and identity
   is not strictly separated from content as the constitutional principle
   requires.
2. **Three-identity model:** EOID (permanent) + Revision ID (sequential,
   human-oriented, independent of content) + Content Hash (integrity,
   dedup, sync). Revision ID's only derivation is from the object's own
   revision sequence, never from content bytes.

## Selected Solution

Three-identity model, as specified in
[02-object-storage-architecture.md](../architecture/02-object-storage-architecture.md)
§1-3: `RevisionID(EOID, n) = <EOID>.r<n>`, sequential per object, with
`content_hash` and `previous_revision_id` stored as separate fields on the
Revision record rather than folded into the Revision ID itself.

## Rationale

- Directly satisfies the constitutional requirement that identity is
  never derived solely from content hashing.
- Makes "revision 7 of this component" a stable, human-usable reference
  that survives even a content revert to earlier bytes — the RC1 design
  would produce a hash collision-adjacent ambiguity in that exact case
  (identical content in two different revisions looks structurally
  similar in a hash-derived scheme, forcing consumers to consult chain
  metadata to disambiguate what should be self-evident from the ID).
- Keeps tampering detection where it belongs: Content Hash mismatches
  catch content tampering; Commit DAG hash chaining
  ([08-integrity-verification-strategy.md](../architecture/08-integrity-verification-strategy.md))
  catches history tampering. Revision ID no longer needs to double as an
  integrity mechanism, so it's free to be simple and sequential.

## Consequences

- `RevisionID` is no longer independently verifiable as a hash — Content
  Hash and the Commit DAG carry the full integrity burden instead,
  requiring implementers to no longer treat "recompute the Revision ID"
  as a verification step (it was never sound as one on its own even
  under RC1, since `commit_id` inside the hash created a circular
  dependency between commit construction and revision hashing that RC1
  did not fully resolve — RC2 removes this ambiguity entirely).
- Every document referencing `RevisionID = Hash(...)` needed correction;
  tracked across
  [02-object-storage-architecture.md](../architecture/02-object-storage-architecture.md),
  [08-integrity-verification-strategy.md](../architecture/08-integrity-verification-strategy.md),
  and this ADR's note on ADR-0002.
- Merge behavior for Revision ID allocation (which side's sequence "wins"
  the next integer) is specified in
  [04-branch-architecture.md](../architecture/04-branch-architecture.md)
  §3 and does not require a new mechanism — the merge commit simply
  allocates the next integer per object on the reconciled branch.

## Future Considerations

If a future need arises for revision identities to be globally unique
across repositories (e.g. cross-repository revision references), the
recommended path is `<repository_id>.<eoid>.r<n>` as a compound reference,
not a redesign of the per-object sequential scheme itself.
