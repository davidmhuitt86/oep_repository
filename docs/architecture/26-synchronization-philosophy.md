# Synchronization Philosophy

Status: Draft v0.1 (Sprint 0.5) · Designed from first principles per directive — does not assume Git, SQL replication, or any specific cloud topology. See [ADR-0019](../adr/ADR-0019-synchronization-philosophy.md).

## 1. First principles

Two Repository instances with a shared history (a common ancestor commit,
directly or via a shared `repository_id` lineage) synchronize by
exchanging the smallest set of Commits, Objects, and Relationships needed
to bring each side's view of the commit DAG up to date with the other —
nothing more centralized is assumed:

- **No hub required.** Two peers with direct connectivity (LAN,
  point-to-point) can synchronize without any server, satisfying
  peer-to-peer topologies.
- **No specific transport assumed.** Synchronization is defined as an
  exchange of Journal segments and content-addressed Blocks
  ([05-transaction-journal-architecture.md](05-transaction-journal-architecture.md)
  §4, [16-native-container-format.md](16-native-container-format.md)) —
  it can run over a direct socket, an HTTP relay, a sneakernet USB drive
  carrying a delta `.oep` package ([07-package-format-proposal.md](07-package-format-proposal.md)
  §5's `delta` package_kind concept, now expressed via native Blocks), or
  a cloud object store acting as a passive relay. None of these transports
  are assumed by the model itself.
- **No assumption of always-connected.** Offline-first
  ([01-repository-specification.md](01-repository-specification.md)) means
  synchronization is always a discrete, bounded exchange initiated when
  connectivity exists, never a continuously-required background channel.

## 2. What gets exchanged

A sync exchange is, at its core, the same content-addressed reachability
computation Git popularized (which commits/objects does peer B have that
peer A doesn't, and vice versa) — but computed generically over the
Object/Relationship/Commit model, not Git's specific object types:

1. Each side exchanges its Branch Store's current tips
   ("what does *your* `main` point to?", `04-branch-architecture.md`).
2. Each side computes, from its own Commit DAG, which ancestor commits of
   the peer's tips it already has, and which it's missing (a "want list"
   — the same fundamental negotiation Git's `have`/`want` protocol
   performs, generalized).
3. Missing Commit/Object/Relationship/Attachment Blocks are transferred,
   content-addressed, verifiable independently as they arrive
   ([19-streaming-model.md](19-streaming-model.md) §6).
4. Each side updates its own Branch Store entries once the transferred
   history is fully verified — never partially, per §3's atomicity note
   below.

## 3. Atomicity of a sync exchange

A sync exchange is itself a Transaction
([25-repository-transactions.md](25-repository-transactions.md)): either
every transferred Commit's dependencies verify and the Branch Store update
is applied, or nothing is applied — no partial history is ever left
locally reachable from a Branch pointer. This is what keeps
`oep.repo.Synchronization` compatible with the Repository's general
crash-safety guarantees rather than being a special unsafe path.

## 4. Peer-to-peer vs. enterprise topologies

The same exchange primitive (§2) supports multiple topologies without a
protocol-level distinction between them:

| Topology | How it's just "sync" |
|---|---|
| Peer-to-peer | Two individual Repository instances sync directly with each other, no third party involved. |
| Enterprise (hub-and-spoke) | An organization designates one Repository instance as a conventional "primary" and has every other instance sync against it — a policy/convention (which peer initiates, which is treated as authoritative for conflict tie-breaking, §5) layered on top of the same exchange primitive, not a different primitive. |
| Partial sync | A peer requests only a subset (e.g. one branch, or commits after a given checkpoint) — the same want-list negotiation (§2 step 2), just scoped, directly reusing the shallow/delta package concepts from [07-package-format-proposal.md](07-package-format-proposal.md) §5. |
| Offline/sneakernet | The "exchange" is mediated by a physical `.oep` delta package instead of a live connection — same content, different transport, no protocol change required. |

`oep.repo.DistributedSynchronization` ([22-repository-capability-model.md](22-repository-capability-model.md))
specifically advertises support for the peer-to-peer/multi-hub case (no
single designated authority), distinct from basic
`oep.repo.Synchronization` (which only requires supporting the exchange
primitive with at least one counterparty, however topology is arranged
above it).

## 5. Conflict detection

A conflict is detected structurally, not semantically: two branches (or
one branch and an incoming peer's view of it) have each advanced past a
common ancestor commit with incompatible `head_commit` values that are
not one a strict descendant of the other. Detection is exactly the
Commit DAG ancestry check already implicit in
[04-branch-architecture.md](04-branch-architecture.md) — sync introduces
no new detection mechanism, only a new *trigger* for running it (an
incoming peer's claimed branch tip, rather than a local merge attempt).

## 6. Conflict resolution and deterministic convergence

The Repository layer's obligation stops at **detection** and at
**recording the resolution as a fact**, consistent with
[04-branch-architecture.md](04-branch-architecture.md) §3: resolution
logic itself (which side wins, how content is reconciled) is a
Capability-level concern, not hardcoded. What synchronization *does*
guarantee, independent of which resolution policy a Capability applies:

- **Deterministic convergence**: given the same set of peers exchanging
  the same commits in any order, and the same resolution policy applied
  to the same conflicts, every peer converges to the identical resulting
  commit DAG and identical `commit_id` values — because commit identity
  is content-derived ([03-commit-architecture.md](03-commit-architecture.md)
  §2), convergence is automatically verifiable (two peers claiming to be
  "synced" can simply compare branch-tip hashes).
- No sync exchange ever silently drops a commit either side had — a
  conflict always produces an explicit merge commit or an explicit
  ConflictPending state ([24-repository-state-machine.md](24-repository-state-machine.md)
  §2), never a silent "last write wins" overwrite at the Repository layer
  (a Capability MAY implement last-write-wins as its resolution *policy*,
  but it does so by producing an explicit, auditable merge commit
  recording that choice — not by discarding history).

## 7. Explicit non-assumptions

Per directive, this model does not assume: a central Git-style remote
convention (`origin`, `upstream`); SQL-style row-level replication logs;
any specific cloud vendor's replication primitives (e.g. DynamoDB streams,
S3 replication). Any of these MAY be used to *implement* the transport
layer under a specific Storage Provider, but none of them are visible in,
or required by, the synchronization model itself.
