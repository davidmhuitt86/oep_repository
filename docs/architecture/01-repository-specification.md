# Repository Specification (Normative)

Status: Draft v0.1 · Defines the on-disk-agnostic layout, manifest, and lifecycle every conformant implementation must expose.

This document uses **MUST / SHOULD / MAY** in the RFC 2119 sense.

## 1. Repository identity

Every Repository instance has a **Repository ID** (UUIDv7, see
[ADR-0002](../adr/ADR-0002-object-identity.md) for the rationale behind
UUIDv7 over UUIDv4/ULID/sequential IDs) assigned at `init` time and never
changed. Two Repositories with the same Repository ID and the same tip
commit hash on the same branch MUST be considered the same engineering
history, regardless of which storage provider or physical location holds
them — this is what makes clone/sync/package round-trips meaningful.

## 2. Logical layout

The layout below is a **logical** structure. A conformant Provider MAY
implement each area as a directory tree, a set of database tables, a set of
object-store prefixes, or anything else — see
[06-storage-provider-abstraction.md](06-storage-provider-abstraction.md).
The names below are the vocabulary the rest of this spec uses; they are not
directory names except in the reference Filesystem Provider.

```
<repository root>
├── manifest                # repository identity + format version (§3)
├── config/                 # provider bindings, policies, local settings
├── objects/                # content-addressed Engineering Object revisions
├── relationships/          # relationship index over the object graph
├── events/                 # append-only event log
├── commits/                # content-addressed commit DAG
├── branches/               # named mutable refs -> commit ID
├── journal/                # write-ahead transaction journal
├── index/                  # derived, rebuildable search/query indexes
├── attachments/             # large binary content, content-addressed
├── integrity/               # Merkle roots, signature chain, verification cache
└── workspace/                # local staging area, never synchronized
```

Everything under `index/` and `workspace/` is a **cache**: it MUST be fully
reconstructible from `objects/`, `relationships/`, `events/`, `commits/`,
and `branches/` alone. Deleting `index/` and `workspace/` and rebuilding
them MUST reproduce identical query results. This is what keeps the
Repository "not a database" — the database-like parts are disposable
projections, not sources of truth.

## 3. Manifest

The manifest is the single fixed entry point of a Repository. It MUST
contain, at minimum:

| Field | Type | Description |
|---|---|---|
| `format_version` | semver string | Version of this specification the Repository conforms to |
| `repository_id` | UUIDv7 | Permanent identity, set once at `init` |
| `created_at` | RFC 3339 timestamp | Creation time |
| `hash_algorithm` | enum | Primary content hash algorithm (`blake3` default, see [08](08-integrity-verification-strategy.md)) |
| `default_branch` | string | Name of the branch opened by default (`main` by convention, not by requirement) |
| `provider_bindings` | map | Which Storage Capability provider backs each store area (see [06](06-storage-provider-abstraction.md)) |

The manifest MUST be readable without interpreting any other store — it is
the only file/record a tool is allowed to assume exists without first
knowing `format_version`.

## 4. Repository lifecycle

### 4.1 Initialize

`init` MUST:
1. Reject if a Repository already exists at the target location (no silent overwrite).
2. Generate `repository_id`.
3. Write the manifest with the current `format_version`.
4. Create an empty journal, an empty commit store, and a single branch
   (`default_branch`) whose ref is unset (no commits yet — an "unborn"
   branch, matching Git's own bootstrap problem and solution).
5. Record an `init` Event as the first Journal entry (see
   [05](05-transaction-journal-architecture.md)) so that even repository
   creation is auditable.

### 4.2 Open

`open` MUST:
1. Read the manifest and verify `format_version` compatibility. An
   implementation MUST refuse to open a newer major format version it does
   not understand, and SHOULD open older minor versions read/write.
2. Replay any uncommitted journal entries (crash recovery, see
   [05](05-transaction-journal-architecture.md) §3).
3. Verify structural integrity at the level configured by policy (see
   [08](08-integrity-verification-strategy.md) §5) — Milestone 1 requires
   at minimum Level 0 (manifest + journal consistency) on every open.

### 4.3 Mutate

All mutation (object creation/modification, relationship creation, commit,
branch move) MUST go through the Transaction Journal — there is no direct
write path to `objects/`, `commits/`, or `branches/` that bypasses it. This
is binding, not optional, because it is the only mechanism that gives the
Repository crash safety and deterministic replay.

### 4.4 Validate

`validate` (exposed to users as `oep verify`, see
[09](09-cli-specification.md)) checks structural, content, and history
integrity without mutating anything. It MUST be safe to run concurrently
with reads and MUST NOT require exclusive locks beyond what an individual
store read requires.

## 5. Conformance

An implementation is conformant if, given the same ordered sequence of
Operations against an empty Repository:

1. It produces the same `repository_id`-independent content hashes for
   every Engineering Object revision.
2. It produces the same commit IDs (commit ID is a hash over normatively
   ordered, canonically serialized commit contents — see
   [03](03-commit-architecture.md) §2).
3. Its `oep verify` reports the same integrity status.

Determinism is deliberately scoped to *content* — implementations are free
to differ in on-disk representation, index structures, and performance
characteristics, as long as the canonical serialization used for hashing
is identical.
