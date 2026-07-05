# Branch Architecture

Status: Draft v0.1 · Defines branches, refs, and merge semantics at the Repository layer.

## 1. A branch is a named, mutable pointer

```
Branch {
  name          : string           // "main", "experimental/thermal-v2", "oem/acme-variant"
  head_commit   : CommitID?        // null only for an "unborn" branch with no commits yet
  purpose_tag   : enum?            // Production | Experimental | Education | Simulation | OEMVariant | Research | custom
  protection    : ProtectionPolicy? // capability-defined, see 06
  created_at    : RFC3339
  created_from  : { branch: string, commit: CommitID }?   // provenance of the branch point
}
```

Branches live in the Branch Store ([01](01-repository-specification.md)
§2), separate from the Commit Store. Moving a branch pointer is itself a
Journaled transaction ([05](05-transaction-journal-architecture.md)) — it
is a mutation like any other, not a special case.

## 2. Branch names carry engineering meaning, not just workflow meaning

Unlike a typical software VCS where branch names are workflow labels, OEP
branches per the constitutional model represent **engineering evolution
lines**: `Production`, `Experimental`, `Education`, `Simulation`,
`OEMVariant`, `Research` are first-class purposes, not conventions. The
`purpose_tag` field lets tooling (and eventually policy) reason about a
branch's role without parsing its name. Custom purposes are allowed —
this is an open enum, matching the platform's extensibility tenet.

## 3. Merge semantics

The Repository defines merge at the structural level only:

1. A merge commit has 2+ `parent_commit_ids`.
2. For each Engineering Object touched on more than one side, the merge
   commit's `object_changes` entry must reference a revision whose
   `previous_revision_id` chain is reachable from *both* parent commits'
   view of that object, OR a new revision explicitly recorded as a
   **conflict resolution** (a revision whose payload metadata references
   both source revisions it reconciles).
3. The Repository does not decide *how* to reconcile conflicting content —
   that is an Operation implemented above the Repository (e.g. Engineering
   Studio's merge UI, or an automated reconciliation Capability). The
   Repository's job is to make sure the result is still a valid,
   fully-chained, verifiable commit.

This mirrors the general philosophy: the Repository stores the *outcome*
of engineering decisions (what the merged state is) as an immutable,
verifiable fact; it does not embed the decision-making logic itself, which
would need to evolve independently of the storage format.

## 4. Branch protection is a Capability, not a hardcoded rule

`ProtectionPolicy` (e.g. "require signed commits," "require all
validation_results to pass," "no force-move of head_commit") is defined and
enforced by a Capability the Repository calls out to at commit/branch-move
time, not baked into the Repository core. Rationale: what "Production"
should require will differ by organization and will change over decades;
hardcoding it into the storage format would violate the technology-neutral,
long-term-standardization tenet.

**Alternative considered:** enforce protection rules directly in Repository
core (simpler initial implementation). Rejected for Milestone-1-and-beyond
architecture, though Milestone 1 itself ships with zero protection
policies configured (open by default) — see
[ADR-0005](../adr/ADR-0005-policy-as-capability.md).

## 5. Deleting and renaming branches

- Renaming a branch changes only the Branch Store entry; no commit is
  reachable-but-orphaned as a result, since commits are addressed by hash,
  not by branch name.
- Deleting a branch removes the pointer, not the commits. A commit
  unreachable from any branch remains in the Commit Store until an
  explicit, auditable garbage-collection Operation runs (out of scope for
  Milestone 1 — see [09](09-cli-specification.md) open items). This is a
  deliberate divergence from typical VCS "aggressive GC" behavior: an
  engineering record with 20+ year retention expectations should never
  silently lose data because a branch pointer moved.

## 6. Unborn branches and the bootstrap case

`init` creates exactly one branch with `head_commit = null`
([01](01-repository-specification.md) §4.1). The first commit on that
branch has 0 parents and, on being committed, sets `head_commit` for the
first time. This is the only point in a Repository's life where a branch
move is not "replace commit A with commit B" but "set commit B for the
first time" — the CLI and any Capability consuming Branch state must treat
`head_commit: null` as a valid, distinct state, not an error.
