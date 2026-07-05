# ADR-0011: Package / Repository / Object Store as three distinct layers

Status: Accepted (RC2)

## Context

RC1 described the Repository as a single undifferentiated system owning
objects, relationships, events, commits, branches, journal, and packaging
together. Sprint 0 review identified this as hidden coupling risk: three
concerns with different lifecycles (distribution, version history, data
model) sharing one conceptual box.

## Problem Statement

Should packaging, version history, and object/relationship/event storage
remain one architectural concept ("the Repository"), or be split into
explicitly layered concerns with a defined dependency direction?

## Alternatives Considered

1. **Single undifferentiated Repository (RC1).** Simplest to describe
   initially, but makes it easy for a future feature (e.g. a packaging
   optimization) to accidentally assume something about object storage
   internals, or for a storage change to leak into packaging behavior,
   with no architectural boundary to catch it.
2. **Three explicit layers — Engineering Package, Engineering Repository,
   Engineering Object Store — each depending only downward.**

## Selected Solution

Three explicit layers, specified in
[10-layered-architecture.md](../architecture/10-layered-architecture.md).

## Rationale

- Matches the actual rate-of-change boundaries: packaging format evolves
  with distribution needs, version history evolves with collaboration
  workflow needs, object storage evolves with the engineering data model.
  Coupling them means a change to one forces review of all three.
- Makes the "Repository does NOT do rendering/simulation/AI/..." boundary
  (00-repository-architecture.md §4) sharper by giving the Repository
  itself internal boundaries, not just an external one.
- Directly enables a future relaxation (e.g. one Object Store shared
  across multiple Repository histories for Engineering Exchange) without
  an architecture rewrite — only a cardinality assumption isolated to one
  layer needs to change.

## Consequences

- Every architecture document must be attributable to exactly one layer;
  ambiguous documents (spanning layers) must be split or explicitly
  scoped, which this ADR's companion document
  (10-layered-architecture.md §3) begins doing for the existing Sprint 0
  document set.
- No implementation is required to physically separate the three layers
  into different deployable components — the requirement is interface
  discipline (§4 of the layering document), not physical separation.

## Future Considerations

If Engineering Exchange (a later-phase component) requires genuinely
decoupled Object Store sharing across Repository histories, that work
should be scoped as "relax the 1:1 cardinality assumption noted in
10-layered-architecture.md §3," not as a new architectural layer.
