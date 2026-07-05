# ADR-0002: UUIDv7 permanent identity + hash-chained revisions

Status: Accepted

## Context

Engineering Objects need a permanent identity distinct from their content
(02-object-storage-architecture.md §1-3), mintable offline, without
central coordination, by any node, forever.

## Problem Statement

What identity scheme gives permanent, collision-free, offline-mintable
Object IDs while remaining time-orderable enough for practical indexing,
and standardized enough to trust for a multi-decade format?

## Alternatives Considered

1. **Sequential integers** — requires central allocation; breaks
   offline-first and makes independently-created objects on two offline
   branches collide on merge.
2. **UUIDv4** — offline-safe, no collisions in practice, but purely random:
   poor index locality, no time information recoverable from the ID.
3. **ULID** — offline-safe and time-sortable, but is a community
   specification, not an IETF standard, weakening the 20-year
   standardization argument.
4. **UUIDv7** — offline-safe, time-sortable (Unix ms timestamp + random
   bits), standardized as IETF RFC 9562.

## Selected Solution

UUIDv7 for Object IDs. Revisions are separately identified via a hash
chain: `RevisionID = Hash(OID || revision_index || content_hash || previous_revision_id || commit_id)`.

## Rationale

- No central allocator needed — satisfies offline-first.
- Time-sortable — improves index locality for large repositories without
  a separate sequence.
- Backed by an IETF RFC — best available long-term standardization
  argument among the alternatives.
- Separating identity (UUIDv7, permanent) from revision (hash chain,
  append-only) directly implements the constitutional principle that
  identity never changes while content does.

## Consequences

- OIDs are 128 bits — larger than a sequential integer, negligible cost at
  engineering-data scale.
- Revision chaining ties every content change to the exact commit that
  produced it, which the Commit Store must reference consistently
  (03-commit-architecture.md §2).
- Independent implementations must use an RFC 9562-conformant UUIDv7
  generator to avoid subtly non-standard IDs; this is a conformance test
  item.

## Future Considerations

If a future IETF revision to UUIDv7 changes bit layout in a
backward-incompatible way, existing OIDs remain valid (they are opaque
128-bit values once minted) — only new-ID generation logic would need
updating, not stored data.
