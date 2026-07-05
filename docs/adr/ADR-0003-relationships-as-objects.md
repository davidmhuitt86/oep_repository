# ADR-0003: Relationships are Engineering Objects, not a separate store

Status: Accepted

## Context

The platform's constitutional model requires everything to be an
Engineering Object connected through typed relationships. A design
decision was needed on whether Relationships get their own storage
subsystem and identity space, or reuse the Object model.

## Problem Statement

Should Relationships be a first-class Engineering Object (type =
`Relationship`), or a dedicated data structure with its own identity and
storage?

## Alternatives Considered

1. **Dedicated relationship store** with its own ID scheme, optimized
   directly for graph traversal (e.g. adjacency-list-native storage).
2. **Relationships as Engineering Objects**, with a derived, rebuildable
   Relationship Index for traversal performance.

## Selected Solution

Relationships are Engineering Objects of type `Relationship`. Traversal
performance is provided by a derived Relationship Index
(02-object-storage-architecture.md §5), which is rebuildable and never a
source of truth.

## Rationale

- Keeps the primitive set closed at five concepts
  (00-repository-architecture.md §3) — a second identity/storage scheme
  for relationships would be a sixth primitive with no compositional
  justification.
- Relationships automatically inherit revision history, content addressing,
  and the ability to be the subject of further relationships (e.g. "this
  connection was validated by that Rule") for free.
- A dedicated store optimized for traversal is still achievable — as an
  Index, which is explicitly allowed and expected to be optimized however
  a Provider likes, without becoming a second source of truth.

## Consequences

- Relationship traversal must go through the Index rather than the Object
  Store directly for performance-sensitive access; this is a documented
  expectation, not an accident.
- Every Provider needs a reasonably efficient way to answer "all
  relationships touching OID X," which the Index capability
  (06-storage-provider-abstraction.md §2.4) exists specifically to serve.

## Future Considerations

If graph-query performance at very large scale (millions of relationships)
proves the generic Index insufficient, the fix is a smarter Index
implementation or a specialized Index Provider — not a new storage
primitive, preserving this ADR's decision.
