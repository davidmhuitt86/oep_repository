# ADR-0012: Elevate the Five-Primitive Rule to a constitutional principle

Status: Accepted (RC2) — amends [ADR-0009](ADR-0009-closed-primitive-set.md)'s standing, not its content

## Context

ADR-0009 established that the platform has five foundational primitives
(Engineering Object, Relationship, Operation, Event, Capability) and that
new concepts should compose from them rather than add a sixth. As an
ordinary ADR, this rule could in principle be superseded by a later ADR
with no higher bar than any other architectural decision — inconsistent
with how central this rule is to every other document in the set.

## Problem Statement

Should the closed-primitive-set rule remain an ordinary ADR (supersedable
like any other), or be elevated to constitutional status, requiring an
explicit amendment process rather than routine supersession?

## Alternatives Considered

1. **Leave as ordinary ADR-0009.** Simpler process, but understates how
   load-bearing this rule is — nearly every other document in this set
   (Commits, Branches, Journal, Relationships) justifies its design by
   appeal to this rule, which is a strong signal it belongs at the same
   level as the philosophy document itself.
2. **Elevate to a constitutional principle**, sitting alongside "the
   specification is the source of truth" and "everything is an Engineering
   Object" as a standing rule that requires the same weight to change as
   those do.

## Selected Solution

Elevated to constitutional status, recorded in
[00-repository-architecture.md](../architecture/00-repository-architecture.md)
§3. The five primitives named in ADR-0009 are unchanged; what changes is
the process required to add a sixth: it now requires an explicit
constitutional amendment (a dedicated ADR arguing composition failure
against all five primitives, reviewed at the same level as this
amendment), not a routine architectural ADR.

## Rationale

- The rule is invoked as justification throughout the document set
  (ADR-0003 relationships-as-objects, ADR-0004 commit/event layering,
  ADR-0005 policy-as-capability) — its de facto importance already
  exceeded its de jure standing as an ordinary, supersedable ADR.
- A 20+ year specification needs a small number of rules that later
  contributors cannot casually erode through routine decisions; this is
  exactly that kind of rule.

## Consequences — Sprint 0 review findings under this rule

As directed, every Sprint 0 document was reviewed against the elevated
rule for unnecessary abstractions, hidden coupling, and primitive-set
violations. Findings and dispositions:

| Finding | Disposition |
|---|---|
| RC1's Revision ID was derived from Content Hash, making revision identity partially content-derived rather than a clean use of the Object primitive's identity/content separation. | Fixed via [ADR-0010](ADR-0010-three-part-identity.md) — Revision ID is now independent, no new primitive introduced. |
| RC1 described "the Repository" as a single undifferentiated box mixing Package, Repository, and Object Store concerns — not a primitive-set violation per se, but a layering ambiguity that made it harder to audit which primitive-set justification applied to which concern. | Fixed via [ADR-0011](ADR-0011-three-layer-separation.md) — layering clarified, no new primitive introduced. |
| `branch_hint` on a Commit record could be read as the Commit primitive absorbing Branch-layer knowledge. | No change needed — already scoped as informative-only, non-authoritative metadata in 03-commit-architecture.md §5; does not constitute a hidden dependency since Branch remains the sole authority. |
| The Package container format decision (ZIP) was made in RC1 without a formal trade-off table against the full set of criteria now required (streaming, signing, random access, forward compatibility, etc.). | Not a primitive-set violation, but flagged for the formal re-evaluation directed separately — see [ADR-0013](ADR-0013-container-format-reevaluation.md). |

No finding required introducing a sixth primitive. This is itself a
positive data point for the rule's durability: three Sprints of real
design work (commits, branches, journal, packaging, provider abstraction)
were all expressible via composition.

## Future Considerations

Any future proposal to add a sixth primitive must be a dedicated ADR that:
(1) names the concept, (2) demonstrates it cannot be expressed as an
Engineering Object type, a Relationship type, or a Capability, (3) is
reviewed with the same scrutiny as this amendment. No such proposal exists
as of RC2.
