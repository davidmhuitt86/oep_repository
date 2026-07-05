# ADR-0009: Five fundamental concepts; no new primitives without exhausting composition first

Status: Accepted

## Context

The constitutional philosophy names four core ideas (Engineering Object,
Operation, Event, Capability) and the architecture adds a fifth compositional
concept, Relationship, built from the first (00-repository-architecture.md
§3). Every subsequent document (Commit, Branch, Journal, Index, Package) is
positioned as a composition of these five rather than a new primitive.

## Problem Statement

As the platform grows over decades, how should the temptation to add a new
fundamental concept for each new feature be resisted, so the core model
doesn't accrete an ever-growing set of special cases?

## Alternatives Considered

1. **No constraint** — let each new subsystem (commits, branches, journal,
   packaging, future simulation/AI integration) introduce whatever concepts
   it needs.
2. **A closed set of five primitives**, with a standing requirement that
   any proposed new concept must first be checked against expressibility as
   an Object, Relationship, Operation, Event, or Capability, and only
   promoted to a true new primitive if all fail.

## Selected Solution

Option 2. Applied concretely in this document set: Commits, Branches, and
the Transaction Journal are all specified as compositions (a Commit is a
grouping of Events plus references to Object/Relationship revisions; a
Branch is a RefStore-backed pointer; the Journal is an AppendLog of
operation intents) — none of them introduce a sixth foundational concept.

## Rationale

- A small, stable core is what makes a 20-year specification tractable —
  every new capability composes on stable ground instead of requiring core
  format changes.
- Matches the explicit "prefer simple core concepts" directive: whenever a
  new abstraction is proposed, check expressibility via the existing five
  first.
- Concretely demonstrated already: Relationships did not need a sixth
  primitive (ADR-0003); Commits did not need a sixth primitive
  (ADR-0004); Branches did not need a sixth primitive
  (04-branch-architecture.md); the Journal did not need a sixth primitive
  (05-transaction-journal-architecture.md).

## Consequences

- Every future design proposal in this project must include an explicit
  "why this isn't expressible as an Object/Relationship/Operation/Event/
  Capability" section before introducing new core vocabulary — a process
  cost, deliberately, to keep the bar high.
- Some designs may end up more indirect than a bespoke concept would be
  (e.g. representing something as an Object with a specific type rather
  than a dedicated structure) — accepted as the cost of long-term
  simplicity.

## Future Considerations

If Phase 2+ work (Simulation Engine, AI Engine integration) surfaces a
concept that genuinely cannot be expressed via the five primitives, that
should produce a new ADR explicitly arguing the composition failure before
any implementation proceeds — never a silent addition.
