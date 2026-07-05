# Transaction Journal Architecture

Status: Draft v0.2 (RC2) · Defines the write-ahead journal every mutation passes through.

**Binding invariant (restated explicitly for RC2):** the journal is
strictly append-only. A repository mutation never overwrites an existing
journal entry — not to correct it, not to compact it, not for any reason.
Every entry, once durably written, is permanent for as long as the segment
containing it exists; the only lifecycle event that removes an entry is
whole-segment retirement under §6, and only after every consumer that
could need it (crash recovery, sync peers) has moved past it. This
invariant is what every other guarantee in this document — recovery,
replay, and auditability — is built on, so it is stated here as a rule,
not inferred from the mechanism.

## 1. Why a journal is mandatory, not an optimization

Four objectives, stated in the directive, none of which are achievable
retroactively once a mutation has already touched the Object/Commit/Branch
stores:

- **Crash recovery** — if the process dies mid-mutation, the stores must
  never be left in a state that is neither "before" nor "after," only
  reachable by replaying a durable log.
- **Rollback** — an in-flight multi-step mutation (e.g. a commit touching
  50 objects) must be entirely abandonable without partial effects.
- **Synchronization** — the journal is the natural unit to ship to a
  replica or peer; replaying someone else's journal segment is how two
  Repositories converge.
- **Deterministic replay / auditability** — the journal is the ground
  truth for "what mutations happened, in what order," independent of
  whatever the derived stores currently look like.

Because of this, the journal is not a performance optimization layered on
top of the "real" writes — it **is** the authoritative record of intent,
and the Object/Commit/Branch/Index stores are the (rebuildable, in the case
of Index) *application* of that record.

## 2. Journal entry structure

```
JournalEntry {
  sequence      : uint64            // monotonic per-repository, gap-free
  entry_id      : Hash(canonical(entry \ entry_id))
  operation     : enum               // CreateObjectRevision | CreateRelationship |
                                      // WriteCommit | MoveBranch | InitRepository | ...
  payload       : bytes              // canonical serialization of the operation's data
  payload_hash  : Hash(payload)
  prev_entry_id : EntryID?           // chains entries, same technique as revisions/commits
  status        : enum               // Pending | Applied | Aborted
  written_at    : RFC3339
}
```

Every entry is appended, hashed, and chained before any corresponding write
happens to `objects/`, `relationships/`, `commits/`, or `branches/`. Only
after the entry is durably written does the Repository apply it to the
target store and mark it `Applied`.

## 3. Crash recovery protocol

On `open` ([01](01-repository-specification.md) §4.2):

1. Scan the journal from the last known `Applied` checkpoint forward.
2. For each `Pending` entry found:
   - If the corresponding store write is verifiably absent, re-apply the
     entry (redo).
   - If the corresponding store write is verifiably present but the entry
     wasn't marked `Applied` (crash occurred between write and mark),
     mark it `Applied` (idempotent redo — operations are designed to be
     safe to apply twice given the same entry).
   - If content hashes don't match what the entry recorded, treat as
     corruption: refuse to open in read-write mode, surface to `oep verify`.
3. Any entry still `Pending` after a configurable recovery timeout with no
   way to safely determine its outcome is marked `Aborted`, and the
   mutation it represented is treated as never having happened. This
   requires every multi-store mutation (e.g. "write object revision" +
   "update branch head") to itself be journaled as a single logical entry,
   or as a small ordered group with the same transaction id — partial
   application across entries is not permitted (see §5).

## 4. Journal as the synchronization primitive

Two Repositories with a common ancestor state can converge by exchanging
journal segments (entries since a known `sequence`/`entry_id` checkpoint)
rather than diffing entire stores. This is the mechanism referenced in
[00-repository-architecture.md](00-repository-architecture.md) as the
future basis for Engineering Exchange sync — out of scope to fully design
in Phase 1, but the journal's append-only, chained, replayable structure
is deliberately chosen so that future design does not require reshaping
this document.

## 5. Transaction grouping

A single Operation (e.g. "commit") frequently implies multiple physical
store writes (N object revisions + M relationship revisions + 1 commit
record + 1 branch move). These are journaled as one **Transaction**: an
ordered group of entries sharing a `transaction_id`, with an explicit
terminal `TransactionCommitted` entry. Recovery treats an incomplete
Transaction (no terminal entry found) as fully `Aborted`, regardless of how
many of its individual entries were durably written — this is what gives
commit atomicity without requiring the underlying Storage Provider to
support multi-key ACID transactions itself (see
[06-storage-provider-abstraction.md](06-storage-provider-abstraction.md)
§4, where this matters for providers like a plain object store that has no
native transaction concept).

## 6. Retention and rotation

Journal segments older than the oldest state any open Repository handle or
replication peer still needs MAY be archived or compacted, but never
silently deleted while `Pending`/unconfirmed-synced entries exist within
them. Segment rotation is a Provider-level implementation detail; the
logical journal (as a chained sequence) is what implementations must
preserve, not any particular file boundary. Rotation removes whole
segments; it never rewrites or truncates an entry within a segment still
in use — consistent with the append-only invariant stated at the top of
this document.

Deeper operational concerns — physical segment layout, compaction
strategy, and long-term maintenance procedures for a journal that has been
running for years — are specified in
[12-repository-internals-lifecycle-and-maintenance.md](12-repository-internals-lifecycle-and-maintenance.md),
which treats the journal as one of several long-lived stores needing a
maintenance strategy.

## 7. What the journal is not

The journal is not a substitute for the Commit DAG's history — it is
short-to-medium-lived operational infrastructure (crash safety, sync),
while Commits are the permanent, long-term engineering record. An
implementation MAY compact/discard journal entries older than the oldest
point any consumer still needs, as long as every `Applied` transaction it
represents has already been durably reflected in the long-lived stores —
Commits, Branches, Objects — which are never compacted away.
