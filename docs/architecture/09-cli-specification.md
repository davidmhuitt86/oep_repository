# Initial Command-Line Interface Specification — Milestone 1

Status: Draft v0.1 · This is the only user-facing surface required for Milestone 1. No GUI.

## 1. Scope

Milestone 1 must let a developer, from the command line alone: initialize
a repository, open it, create Engineering Objects, create typed
relationships, commit changes, view history, and reconstruct repository
state at any commit. Everything below maps directly to those seven
capabilities plus the packaging and verification mechanisms already
specified.

## 2. Command surface

```
oep init <path> [--hash-algorithm blake3|sha256]
    Creates a new Repository at <path>. Refuses if one already exists
    (01-repository-specification.md §4.1).

oep open <path>
    Opens and validates (Level 0) a Repository, prints manifest summary
    and current branch/head status.

oep status
    Shows current branch, head commit, and any staged-but-uncommitted
    Operations pending in the workspace area.

oep object create --type <type> --data <file|-> [--attr key=value ...]
    Stages a new Engineering Object (assigns OID, revision 0) with the
    given typed payload. Does not commit — staged in workspace
    (01-repository-specification.md §2) until `oep commit`.

oep object show <oid> [--revision <revision-id>] [--history]
    Displays an object's current or specified revision; --history lists
    the full revision chain (02-object-storage-architecture.md §3).

oep relationship create --type <type> --from <oid> --to <oid> [--attr key=value ...]
    Stages a Relationship object (02-object-storage-architecture.md §5).

oep commit -m <message> [--sign]
    Commits all staged Operations as one Commit
    (03-commit-architecture.md), journaled as a single Transaction
    (05-transaction-journal-architecture.md §5). --sign attaches a
    detached signature using the caller's configured Identity.

oep log [--branch <name>] [--limit N] [--oid <oid>]
    Walks the commit DAG from a branch tip (default: current branch).
    --oid filters to commits touching a specific Engineering Object.

oep branch list
oep branch create <name> [--from <branch|commit>] [--purpose <tag>]
oep branch switch <name>
    Branch operations (04-branch-architecture.md). `create` records
    `created_from` provenance.

oep checkout <commit-id>
    Reconstructs full repository state (materialized objects and
    relationships as of that commit) into a read-only view — this is the
    "reconstruct repository state" requirement, and MUST produce output
    identical regardless of which Storage Provider backs the Repository
    (01-repository-specification.md §5, conformance).

oep verify [--level 0|1|2]
    Runs integrity verification (08-integrity-verification-strategy.md
    §5). Default: level 0.

oep pack <output.oep>
oep unpack <input.oep> <path>
    Package export/import (07-package-format-proposal.md).
```

## 3. Explicitly out of scope for Milestone 1

- Merge conflict resolution UX (structural merge model exists,
  04-branch-architecture.md §3; no CLI merge tool yet).
- Remote synchronization (`oep push`/`pull` or equivalent) — deferred to
  the Engineering Exchange integration phase.
- Query language / `oep search` beyond `--oid` filtering on `log`.
- Branch protection policy configuration (`oep branch protect ...`) —
  the mechanism is specified (04-branch-architecture.md §4) but no
  Milestone-1 command surface is required to exercise it.
- Garbage collection of unreachable commits (04-branch-architecture.md §5).

These are omitted from *scope*, not from the format — a Milestone 1
Repository is not a "lite" format; later milestones only add commands, they
never require a migration of Milestone-1-created repositories.

## 4. Exit criteria for Milestone 1

Milestone 1 is complete when the following sequence, run against the
reference Filesystem Provider, produces a Repository that:

1. Passes `oep verify --level 2`.
2. Round-trips through `oep pack` → `oep unpack` with byte-identical
   `oep verify` results before and after.
3. Produces identical `commit_id` values when the same sequence of
   commands is run against a second Storage Provider (e.g. SQLite),
   demonstrating the conformance property in
   [01-repository-specification.md](01-repository-specification.md) §5.

```
oep init demo
cd demo
oep object create --type oep.core.Component --data component.json
oep object create --type oep.core.Component --data sensor.json
oep relationship create --type connects_to --from <oid1> --to <oid2>
oep commit -m "Initial wiring"
oep log
oep checkout <commit-id>
oep verify --level 2
oep pack demo.oep
```
