# ADR-0001: Content-addressable storage for Engineering Objects

Status: Accepted

## Context

Engineering Object revision payloads (02-object-storage-architecture.md)
need a storage addressing scheme that works identically across every
future Storage Provider and supports integrity verification without an
external index.

## Problem Statement

How should revision content be addressed and stored so that identical
content is deduplicated, corruption is self-evident, and the addressing
scheme itself never depends on a particular storage technology?

## Alternatives Considered

1. **Location-addressed storage** (path or row-ID as address). Simple, but
   ties addressing to a specific Provider's layout and gives no free
   integrity check — a corrupted file at the right path looks valid.
2. **UUID-per-revision, content stored separately from its address.**
   Avoids Provider coupling but requires a separate integrity mechanism
   and gains no deduplication.
3. **Content-addressed storage** (address = hash of content).

## Selected Solution

Content-addressed storage: the address of any stored revision payload or
attachment is its cryptographic hash.

## Rationale

- Deduplication is automatic and free.
- Corruption/tampering is self-evident: a Provider returning bytes that
  don't hash to the requested address is provably wrong.
- The addressing scheme is a pure function of content, so it is identical
  across every Provider (02-object-storage-architecture.md §4,
  06-storage-provider-abstraction.md).
- It composes directly into the Merkle structure used for commit and
  history integrity (08-integrity-verification-strategy.md §2) with no
  additional data structure.

## Consequences

- Content is immutable by construction — "editing" content means creating
  a new hash-addressed blob and a new revision, never mutating in place.
  This aligns with the constitutional "nothing silently modified"
  principle without any extra enforcement code.
- Requires every Provider to implement a `BlobStore` capability
  (get/put/has by hash) — a small, easy contract
  (06-storage-provider-abstraction.md §2.1).
- Large attachments need chunking/streaming strategies for Providers with
  size-limited object semantics; deferred to Provider-level implementation
  detail, not a format concern.

## Future Considerations

If a future hash algorithm is adopted, existing content-addressed data
keeps its original hash tag permanently (ADR-0008) — this ADR's decision
does not need revisiting when algorithms evolve.
