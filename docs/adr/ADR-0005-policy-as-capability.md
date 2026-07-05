# ADR-0005: Branch protection and validation enforcement are Capabilities, not core logic

Status: Accepted

## Context

Engineering organizations will want rules like "Production requires signed
commits" or "all validation rules must pass before merging to Production."
These rules will differ by organization and change over decades.

## Problem Statement

Should policy enforcement (branch protection, validation gating) be
implemented directly inside Repository core, or delegated to a Capability
the Repository calls out to?

## Alternatives Considered

1. **Hardcode common policies into Repository core** (e.g. a fixed set of
   protection flags interpreted by the commit/branch-move code path).
2. **Repository core only records outcomes** (`validation_results` on a
   commit, `ProtectionPolicy` on a branch as opaque-to-core data) and
   **delegates enforcement to a Capability**.

## Selected Solution

Policy enforcement is a Capability (04-branch-architecture.md §4,
03-commit-architecture.md §6). The Repository core stores policy
configuration and validation outcomes as data but does not itself decide
whether a commit should be rejected based on organizational policy.

## Rationale

- Policy will legitimately vary by organization and evolve continuously;
  baking specific rules into the storage format would require a format
  version bump every time policy needs change, which is incompatible with
  a specification meant to last decades.
- Matches the constitutional principle that applications request
  Capabilities rather than depending on implementations directly —
  policy enforcement is exactly this kind of pluggable function.
- Keeps the Repository's conformance surface (01-repository-specification.md
  §5) small and stable: conformance is about content hashing and DAG
  structure, not about whose organizational rules happened to be active.

## Consequences

- Milestone 1 ships with no enforced policies (open by default) — the
  mechanism exists but nothing requires exercising it yet.
- Any tool built on the Repository that wants gated branches must supply
  its own policy Capability implementation; the Repository will not do
  this for them out of the box.

## Future Considerations

A reference policy Capability (e.g. "require N signatures," "require all
validation_results = pass") is a reasonable Phase 2 deliverable, but as an
optional add-on module, not a Repository-core change.
