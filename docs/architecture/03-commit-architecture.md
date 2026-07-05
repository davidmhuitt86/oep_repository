# Commit Architecture

Status: Draft v0.1 · Defines what a commit represents, its structure, and the commit DAG.

## 1. A commit is an engineering change, not a file diff

A commit records: which Engineering Objects got new revisions, which
Relationships changed, which Events occurred, and what the outcome of any
validation/rule evaluation was — all as one atomic, signed, immutable unit.
There is no notion of "unstaged file changes" the way a filesystem-based
VCS has; the unit of change is always a coherent engineering action or
batch of actions.

## 2. Commit structure

```
Commit {
  commit_id            : Hash(canonical(Commit \ commit_id))   // self-referential; excluded from its own hash input
  parent_commit_ids    : [CommitID]     // 0 = root commit, 1 = normal, 2+ = merge
  repository_id        : UUIDv7
  branch_hint          : string          // informative only, not authoritative (see §5)
  author               : Identity        // human, service, or AI agent identity
  timestamp            : RFC3339
  message              : string
  object_changes       : [{ oid, from_revision?, to_revision }]
  relationship_changes : [{ relationship_oid, from_revision?, to_revision }]
  event_refs           : [EventID]       // Events this commit groups, in order
  validation_results   : [{ rule_id, outcome, detail_ref }]
  metadata             : map<string, any>   // extensible, capability-defined
  signature            : DetachedSignature?  // see 08-integrity-verification-strategy.md
}
```

`commit_id` is computed over the canonical serialization of every field
except itself and `signature` (the signature is computed over `commit_id`,
so the order is: build commit → hash → sign → attach signature).

## 3. The commit DAG

Commits form a directed acyclic graph via `parent_commit_ids`, exactly as
in Git's model, generalized to engineering objects instead of files:

- **0 parents** — root commit of a Repository (or of an intentionally
  disconnected history, e.g. an imported external dataset).
- **1 parent** — a normal sequential change.
- **2+ parents** — a merge, recording that this commit reconciles
  divergent engineering evolution from multiple branches (see
  [04-branch-architecture.md](04-branch-architecture.md) §3 for merge
  semantics — the Commit format only records *that* a merge happened and
  what the reconciled state is, not the merge algorithm).

Because every commit is content-addressed and its hash depends on its
parents' hashes, the entire history up to any commit reduces to that one
commit's hash — a full Merkle proof of everything that happened, matching
[08-integrity-verification-strategy.md](08-integrity-verification-strategy.md).

## 4. Commits are not Events, they group Events

A subtlety worth being explicit about, since the constitutional philosophy
states "every engineering change is an event": Events are the atomic
record of *what happened* (e.g. `ObjectRevisionCreated`,
`RelationshipEstablished`, `MeasurementRecorded`). Commits are the
*unit of agreement* — a point where a coherent set of Events is bundled,
validated, optionally signed, and made a permanent point in history that
branches and merges can reference.

This two-layer design (Event = fact, Commit = checkpoint) is deliberate:
it lets an implementation batch many fine-grained Events (e.g. a bulk
import of 10,000 measurements) into one Commit without losing per-Event
granularity, and it lets tooling replay Events independently of commit
boundaries for audit or streaming purposes (see
[05-transaction-journal-architecture.md](05-transaction-journal-architecture.md)
§4, synchronization).

**Alternative considered:** treat every Event as its own commit
(finest-grained possible history). Rejected — it would make the commit DAG
enormous and defeat the purpose of a commit as a stable reference point for
branching/merging/tagging; see
[ADR-0004](../adr/ADR-0004-commit-groups-events.md).

## 5. Branch is a pointer, not a commit field

`branch_hint` in the Commit structure is informative metadata only (useful
for tooling and audit trails), never authoritative. The authoritative
mapping from branch name to commit is owned entirely by the Branch Store
([04-branch-architecture.md](04-branch-architecture.md)). This separation
is what allows a commit to be later included in a different branch's
history (e.g. cherry-picked, or the branch renamed) without altering the
commit's identity or hash.

## 6. Validation results are recorded, not enforced, by the Repository

`validation_results` lets a commit carry the outcome of rule evaluation
(a Capability provided by higher layers, e.g. Canon) as part of the
permanent record — "this commit passed these checks at this time." The
Repository itself does not execute rules or block a commit based on
validation outcome; policy enforcement (e.g. "production branch requires
all rules to pass") is a Capability-level concern layered on top, so the
Repository's core commit mechanism stays free of business logic that
would otherwise need to change every time engineering policy changes.
