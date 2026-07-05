# Repository Internals II — Garbage Collection, Repair, Upgrade, and Maintenance

Status: Draft v0.1 (Sprint 0.5) · Architecture only — no production implementation is specified here.

This document covers the long-lived operational lifecycle of a Repository
that has been running for years, complementing the physical/encoding
concerns in
[11-repository-internals-storage-and-encoding.md](11-repository-internals-storage-and-encoding.md).

## 1. Garbage collection

Per [04-branch-architecture.md](04-branch-architecture.md) §5, deleting a
branch removes only the pointer; unreachable commits (and the object
revisions/attachments only reachable through them) remain until an
explicit GC Operation runs. GC is deliberately **never automatic** —
running it is itself a Journaled, auditable Operation
([05-transaction-journal-architecture.md](05-transaction-journal-architecture.md)),
producing an Event recording exactly what was removed, because an
engineering record with multi-decade retention expectations must never
lose data as a side effect of routine use.

GC algorithm (informative, not normative on implementation):

1. Compute the reachable set: every commit reachable from every Branch
   Store entry (including any explicitly retained tags/refs a Capability
   layer may define), and every Object/Attachment revision referenced from
   those commits.
2. Anything content-addressed and not in the reachable set is a GC
   candidate.
3. Candidates are staged (moved to a quarantine area, not deleted) for a
   configurable grace period, then physically removed — giving a recovery
   window for accidental branch deletion, which is the far more common
   failure mode than a genuine need to reclaim space.
4. The GC run itself is recorded as a Commit-adjacent Event so `oep log
   --gc-history` (a plausible future command) can show exactly when and
   what was reclaimed.

## 2. Repository optimization

Optimization (repacking objects per
[11](11-repository-internals-storage-and-encoding.md) §3, rebuilding
indexes for better locality, recompressing with a newer codec) never
changes logical content or any Content Hash — it is purely a physical
rewrite behind the `BlobStore`/`Index` Capability boundary. Optimization
MUST be safe to interrupt (crash mid-optimization must leave the
Repository in a valid, if unoptimized, state) — achieved by writing
optimized structures to a new location and atomically swapping via the
`RefStore` Capability's compare-and-swap primitive
([06-storage-provider-abstraction.md](06-storage-provider-abstraction.md)
§2.3), never by rewriting in place.

## 3. Repository repair

Repair addresses detected integrity failures
([08-integrity-verification-strategy.md](08-integrity-verification-strategy.md)):

| Detected condition | Repair strategy |
|---|---|
| Content Hash mismatch, content recoverable from a replication peer or package backup | Re-fetch and re-verify from a trusted source; never "repair" by re-deriving content from assumptions |
| Journal gap (chain break) with all referenced commits/objects otherwise intact | Journal is operational infrastructure, not the permanent record ([05](05-transaction-journal-architecture.md) §7) — rebuild journal state from the authoritative Commit/Branch/Object stores rather than attempting to reconstruct lost journal entries |
| Commit DAG signature failure | MUST NOT auto-repair silently — surfaced to the user/administrator as a trust decision, since "fixing" a signature failure is indistinguishable from forging one without human judgment |
| Corrupted, unrecoverable content with no available good copy | Recorded as a permanent, explicit gap (a tombstone Event), never silently dropped — the engineering record shows "this content existed and became unrecoverable at time T," not nothing |

Repair is always additive/auditable — it never silently rewrites history
to erase evidence that something went wrong.

## 4. Repository upgrade strategy

The manifest's `format_version`
([01-repository-specification.md](01-repository-specification.md) §3) is
the anchor for upgrades:

- **Minor version upgrades** (new optional fields, new Object/Relationship
  types, new optional Capabilities) require no data migration — older
  data simply doesn't populate the new optional fields.
- **Major version upgrades** (changes to canonical serialization rules,
  hash algorithm defaults, or core structural fields) require an explicit
  **migration Operation**: read under the old `format_version`'s rules,
  re-write under the new rules, producing new content hashes where
  serialization changed. This MUST be a Journaled, auditable, resumable
  Operation — treated as any other large mutation, never a special
  "maintenance mode" that bypasses the journal.
- A Repository MUST refuse to open (read-write) under a major version it
  doesn't recognize rather than guess-parse it
  ([01](01-repository-specification.md) §4.2) — silent misinterpretation
  of a future format is a worse failure mode than a clear refusal.

## 5. Repository diagnostics

`oep verify` ([08](08-integrity-verification-strategy.md) §5) is the
correctness diagnostic. A separate **health diagnostic** surface (a later
milestone, sketched here for architectural completeness) should report,
without requiring full verification:

- Store sizes and growth rate (objects, journal, index) — capacity
  planning signal.
- Journal segment count and oldest un-rotated segment age — sync/backlog
  signal.
- GC candidate volume — how much reclaimable space exists.
- Provider-reported health (e.g. a cloud Provider's own service status) —
  passed through opaquely, never interpreted by Repository core.

## 6. Repository maintenance — putting it together

Long-term maintenance of a Repository is the composition of the above,
run on an operator-chosen schedule, never silently by the Repository
itself:

```
oep gc [--dry-run] [--grace-period <duration>]     # §1
oep optimize                                         # §2
oep verify --level 2                                 # detect (08 §5)
oep repair --from <peer|package>                     # §3, requires a trusted source
oep upgrade --to <format_version>                    # §4
oep diagnostics                                       # §5
```

None of these commands are required for Milestone 1
([09-cli-specification.md](09-cli-specification.md) §3); they are
specified here so that when they are built, they compose with the
already-frozen Milestone 1 format rather than requiring it to change.
