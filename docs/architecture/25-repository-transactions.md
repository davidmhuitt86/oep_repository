# Repository Transactions

Status: Draft v0.1 (Sprint 0.5) · Defines transaction semantics and their integration with the Engineering Event Model.

## 1. Relationship to the Journal and the Event Model

A Transaction is the unit of atomicity already introduced in
[05-transaction-journal-architecture.md](05-transaction-journal-architecture.md)
§5 (an ordered group of journal entries sharing a `transaction_id`,
terminated by a `TransactionCommitted` entry). This document specifies
the semantics of that unit in full, and its integration with Events
([03-commit-architecture.md](03-commit-architecture.md) §4): **a
Transaction is the mechanism; a Commit is one possible durable outcome of
a successful Transaction; the Events a Commit groups are the semantic
content the Transaction made durable.** A Transaction can also complete
without producing a Commit at all (e.g. a Transaction that only creates
staged workspace state, not yet committed) — Transaction atomicity and
Commit creation are related but distinct concepts, and conflating them
was flagged in review as a risk to avoid.

## 2. Atomic operations

The smallest Transaction is a single Operation
([00-repository-architecture.md](00-repository-architecture.md) §3)
producing zero or more Events, journaled as one or more entries under one
`transaction_id`, terminated by `TransactionCommitted`. Atomicity
guarantee: either every entry in the Transaction is durably applied and
the terminal entry is present, or none of the Transaction's effects are
considered to have happened at all (§ [05-transaction-journal-architecture.md](05-transaction-journal-architecture.md)
§5).

## 3. Composite operations

A composite Transaction groups multiple logical steps that must succeed
or fail together — e.g. "create three Objects, establish two Relationships
between them, and commit" as `oep commit` performs
([09-cli-specification.md](09-cli-specification.md)). Composite
Transactions are still one `transaction_id`; there is no partial-success
concept at the Transaction level — a composite Transaction that fails
partway aborts entirely (§5).

## 4. Nested transactions

**Recommendation: no true nested transactions in v0.1; savepoints
within a single Transaction instead.** Rationale: true nested
transactions (independently committable sub-transactions inside a parent)
introduce substantial complexity (partial-commit visibility rules,
nested-rollback semantics) that the Journal's flat
`transaction_id`-grouped-entries model does not need to solve for any
Milestone 1 or near-term use case. A **savepoint** — a named marker within
an in-flight Transaction that a caller can roll back to without aborting
the whole Transaction — covers the practical need (e.g. "undo the last
Relationship I staged, keep the two Objects I already staged") without
the full complexity of independently-committable nested scopes:

```
tx = begin_transaction()
tx.create_object(...)
sp = tx.savepoint()
tx.create_relationship(...)     // turns out to be wrong
tx.rollback_to(sp)               // discards only the relationship
tx.create_relationship(...)     // corrected
tx.commit()
```

## 5. Rollback

Rollback (full-Transaction) discards every journal entry associated with
`transaction_id` and applies none of their effects — implemented as: the
terminal `TransactionCommitted` entry is simply never written, and
recovery (§ [05-transaction-journal-architecture.md](05-transaction-journal-architecture.md)
§3) treats any Transaction lacking that terminal entry as fully aborted,
regardless of how many of its individual entries were durably written
before the rollback decision. This means rollback never requires
"undoing" already-applied store writes — the Journal's replay rule already
guarantees an un-terminated Transaction is invisible.

## 6. Checkpoints

A **checkpoint** is a caller-visible confirmation that a Transaction's
effects up to a given savepoint (§4) are durable and will survive a crash,
*without* ending the Transaction. This differs from a savepoint (a
rollback target) in intent: a checkpoint is about durability confirmation
for a long-running composite Transaction (e.g. a bulk import staging
10,000 measurements, per [03-commit-architecture.md](03-commit-architecture.md)
§4's batching rationale), letting a caller confirm progress without
committing early and losing the "one coherent Commit" semantics the batch
import wants.

## 7. Replay

Given the same ordered Journal (or Journal segment,
[05-transaction-journal-architecture.md](05-transaction-journal-architecture.md)
§4), replaying it against a fresh empty store MUST produce byte-identical
resulting state — this is the same determinism guarantee
[01-repository-specification.md](01-repository-specification.md) §5
requires of the Repository as a whole, restated here as a Transaction-level
obligation: no Transaction may depend on ambient, non-journaled state
(wall-clock time not recorded in the entry, random values not recorded in
the entry, environment-dependent behavior) to determine its effects.

## 8. Failure recovery

Failure during a Transaction falls into the crash-recovery protocol
already specified in
[05-transaction-journal-architecture.md](05-transaction-journal-architecture.md)
§3, restated here as the Transaction-level contract: on reopen, any
Transaction without a `TransactionCommitted` terminal entry is treated as
if `rollback()` (§5) had been called on it, with no further caller
involvement required — recovery is fully automatic for this case,
unlike ConflictPending sync failures ([24-repository-state-machine.md](24-repository-state-machine.md)
§2), which require explicit resolution because they involve two parties'
intent, not just one party's crash.

## 9. Integration with the Event Model — summary

```
Operation requested
   │
   ▼
Transaction begins (transaction_id assigned)
   │
   ▼
Events produced, journaled as entries under transaction_id  ── (§2-3)
   │
   ▼
[optional: savepoints (§4), checkpoints (§6) along the way]
   │
   ├─▶ success: TransactionCommitted entry written
   │       │
   │       ▼
   │   Events optionally grouped into a Commit (03-commit-architecture.md §4)
   │       — grouping into a Commit is a caller decision, not automatic
   │
   └─▶ failure: no terminal entry written → treated as fully rolled back
           on next recovery pass (§8), Events never considered to have
           occurred
```
