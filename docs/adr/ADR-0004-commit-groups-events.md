# ADR-0004: Commits group Events; they are not one-Event-per-commit

Status: Accepted

## Context

The constitutional philosophy states every engineering change is an
Event, and separately that a Commit represents engineering change. These
could be conflated into a single mechanism (each Event is its own commit)
or kept as two layers.

## Problem Statement

Should the finest-grained unit of history (the Event) also be the unit
that branches, merges, and tags operate on (the Commit), or should Commits
group multiple Events into a coherent checkpoint?

## Alternatives Considered

1. **One Event = one Commit.** Simplest mental model; maximal history
   granularity.
2. **Commits group an ordered set of Events**, with Events remaining
   independently replayable.

## Selected Solution

Commits group Events (03-commit-architecture.md §4). A commit references
an ordered list of `event_refs` plus the resulting object/relationship
changes; Events themselves remain individually addressable in the Event
Store.

## Rationale

- A bulk operation (e.g. importing 10,000 measurements) would otherwise
  produce 10,000 commits, making the commit DAG useless as a stable
  reference point for branching, merging, and tagging.
- Keeping Events independently stored still allows fine-grained audit,
  streaming, and synchronization use cases
  (05-transaction-journal-architecture.md §4) without forcing every
  consumer of history to work at commit granularity.
- Mirrors a well-understood, proven pattern (event sourcing with
  checkpoints) rather than inventing new semantics.

## Consequences

- Two data structures (Event Store, Commit Store) instead of one, with a
  defined reference relationship between them.
- Tooling that wants full granularity (e.g. "show me exactly what changed,
  in order, within this commit") reads `event_refs`, not just the
  aggregate `object_changes`.

## Future Considerations

None identified that would require revisiting this decision; the
two-layer model is expected to remain stable as higher layers (Simulation,
AI) add new Operation/Event types, since neither requires changes to the
Commit structure itself.
