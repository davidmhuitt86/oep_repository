# ADR-0015: Repository Capability advertisement instead of version-gated behavior

Status: Accepted (Sprint 0.5)

## Context

Repositories will increasingly support optional behaviors (branching,
signing, sync, multi-user collaboration, archive mode) that vary by
deployment, policy, and format version. Clients need a reliable way to
discover what a specific Repository instance supports before attempting
an operation.

## Problem Statement

Should optional Repository behavior be gated purely by `format_version`,
or advertised explicitly as a set of Capabilities independent of format
version?

## Alternatives Considered

1. **Format-version-only gating.** Simple, but conflates "what bytes can
   this reader parse" with "what behavior is this instance willing to
   perform right now" — a Repository in `ImmutableArchiveMode` or
   `ReadOnlyMode` has no way to express reduced behavior without lying
   about its format version.
2. **Explicit Capability advertisement**, reusing the platform's general
   Capability primitive one level up, applied to Repository instances
   themselves.

## Selected Solution

Explicit Capability advertisement, specified in
[22-repository-capability-model.md](../architecture/22-repository-capability-model.md).

## Rationale

- Separates parsing concerns (format_version) from behavioral/policy
  concerns (capabilities), matching how policy enforcement was already
  kept out of core commit mechanics ([ADR-0005](ADR-0005-policy-as-capability.md)).
- Open capability ID namespace (reverse-DNS, like Object types) avoids a
  central registry bottleneck and lets custom deployments introduce
  organization-specific capabilities without a spec change.
- Makes gated behavior discoverable up front rather than
  trial-and-error (attempt-then-handle-failure), which is friendlier to
  both human tooling and automated agents.

## Consequences

- Every optional Repository behavior introduced from Sprint 0.5 onward
  must be expressed as a capability ID, not a hidden format-version
  assumption.
- Clients must check capabilities before depending on them; this is now
  a conformance expectation for well-behaved clients, not just a
  suggestion.

## Future Considerations

If capability interactions become complex (e.g. capability A requires
capability B), a dependency-declaration mechanism can be added to
`CapabilityFlag` without breaking existing capability IDs.
