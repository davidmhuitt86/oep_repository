# Multi-User Behavior

Status: Draft v0.1 (Sprint 0.5) · Defines concurrent-editing semantics. See [ADR-0020](../adr/ADR-0020-optimistic-concurrency-default.md).

## 1. Concurrency model: optimistic by default

**Recommendation: optimistic concurrency as the default, pessimistic
locking as an opt-in Capability-gated policy, not the core model.**

| Model | How it works here | Trade-off |
|---|---|---|
| **Pessimistic (locking)** | A user must acquire an exclusive lock on an EOID (or a whole Repository) before editing; other users are blocked until released. | Prevents conflicts outright, but doesn't fit offline-first: a lock held by an offline user blocks everyone else indefinitely, which is unacceptable for a platform whose Storage Providers explicitly include in-memory/offline/local-first cases ([06-storage-provider-abstraction.md](06-storage-provider-abstraction.md)). |
| **Optimistic** | Any user may create a new Revision against their last-known state at any time, offline or online; conflicts are detected at sync/merge time ([26-synchronization-philosophy.md](26-synchronization-philosophy.md) §5), not at edit time. | Matches offline-first directly; requires a real conflict-resolution story (§3), which the platform already has via the Commit/Branch/merge model. |

Optimistic concurrency is selected as the default because it is the only
model compatible with the constitutional offline-first requirement
without contradiction — a pessimistic lock cannot be meaningfully enforced
against a party who is, by design, allowed to be disconnected indefinitely.
Pessimistic locking remains available as an **opt-in policy Capability**
(e.g. an organization's Engineering Studio deployment enabling
"soft locks" as an editing-etiquette UI hint, not a Repository-enforced
hard lock) for teams that want it for specific high-contention Object
types — but the Repository core never assumes it is in effect.

## 2. Concurrent editing

Two users editing the same EOID offline, disconnected from each other,
each produce their own next Revision on their own local branch history.
Neither is "wrong" — both are valid, fully-formed engineering changes
until a sync/merge brings them together. This directly reuses the
Revision model ([02-object-storage-architecture.md](02-object-storage-architecture.md))
and Commit DAG ([03-commit-architecture.md](03-commit-architecture.md)) —
no new mechanism is introduced for "concurrent edit," because from the
Repository's perspective two offline users editing the same object are
structurally identical to two branches diverging, which the platform
already models.

## 3. Conflict ownership

When a sync (§ [26-synchronization-philosophy.md](26-synchronization-philosophy.md)
§5-6) detects a conflict, **conflict ownership defaults to both authors
jointly** — the Repository does not pick a "winner" automatically. A
Capability-level policy MAY assign ownership differently (e.g.
"Production branch's existing content always takes precedence over an
incoming merge, subject to human review" for a `Production`-tagged
branch, [04-branch-architecture.md](04-branch-architecture.md) §2), but
the Repository's default behavior is to surface the conflict
(`ConflictPending`, [24-repository-state-machine.md](24-repository-state-machine.md))
rather than silently resolve it in either author's favor.

## 4. Merge semantics

Unchanged from [04-branch-architecture.md](04-branch-architecture.md) §3:
the Repository verifies structural validity of a merge commit; it does
not decide reconciliation content. Multi-user conflict resolution is
therefore the same mechanism as branch merge conflict resolution — a
deliberate simplification: **the platform has exactly one conflict
resolution mechanism** (the merge commit), used whether the divergence
came from two named branches or two offline users editing concurrently.
This is a direct application of the closed-primitive-set discipline
([00-repository-architecture.md](00-repository-architecture.md) §3,
elevated to constitutional status by
[ADR-0012](../adr/ADR-0012-five-primitive-constitutional-amendment.md)) —
multi-user collaboration did not need a sixth primitive or a second
conflict mechanism; it composes entirely from Branches and Commits already
specified.

## 5. Event ordering

Events within a single author's Transaction are strictly ordered (journal
sequence, [05-transaction-journal-architecture.md](05-transaction-journal-architecture.md)
§2). Across two concurrently-editing, disconnected authors, there is
**no global total order** until sync brings their histories together —
this is a deliberate rejection of any assumption that wall-clock
timestamps alone establish a trustworthy cross-author order (clock skew
across disconnected offline nodes makes this unreliable, the same reason
[ADR-0002](../adr/ADR-0002-object-identity.md) rejected timestamp-based
identity). Cross-author causal ordering, where it matters, is established
by the Commit DAG's parent-pointer structure once merged
([03-commit-architecture.md](03-commit-architecture.md) §3), which is a
partial order (a DAG), not a forced total order — correctly reflecting
that concurrent, unrelated changes genuinely have no meaningful "before/
after" relationship to each other until something (a merge) relates them.

## 6. Repository consistency under multi-user load

"Consistency" for a multi-user Repository means: every peer that has
synced to the same set of branch tips sees identical content for every
EOID at those tips (content-hash-verifiable,
[08-integrity-verification-strategy.md](08-integrity-verification-strategy.md)),
not that every peer sees updates from every other peer instantaneously.
This is an explicit choice of **eventual, verifiable consistency** across
disconnected peers, with **strict consistency within a single peer's own
Transaction boundary** ([25-repository-transactions.md](25-repository-transactions.md))
— the same two-tier consistency model distributed systems generally
settle on when strict global consistency would require an always-connected
assumption the platform explicitly rejects.
