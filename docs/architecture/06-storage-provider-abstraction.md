# Storage Provider Abstraction

Status: Draft v0.1 · Defines the Capability contracts that decouple the Repository from any storage technology.

## 1. Principle

The Repository API (Objects, Relationships, Commits, Branches, Journal,
Index) never changes based on what's underneath it. Storage engines are
Capabilities: named, versioned contracts a Provider implements and the
Repository core consumes through a stable interface. No component above
the Repository is ever aware of which Provider is bound.

## 2. Capability contracts

Four minimal contracts, deliberately small, cover everything the
Repository core needs:

### 2.1 `Capability.Storage.BlobStore` (v1)
Content-addressed get/put for immutable bytes.
```
put(content: bytes) -> ContentHash
get(hash: ContentHash) -> bytes
has(hash: ContentHash) -> bool
```
Backs: Object revision payloads, Commit records, Attachments.

### 2.2 `Capability.Storage.AppendLog` (v1)
Strictly ordered, append-only sequence with durable flush semantics.
```
append(entry: bytes) -> sequence: uint64
read_from(sequence: uint64) -> stream<entry>
last_sequence() -> uint64
```
Backs: the Transaction Journal, the Event Store.

### 2.3 `Capability.Storage.RefStore` (v1)
Mutable named pointer with compare-and-swap semantics (required for safe
concurrent branch moves).
```
get(name: string) -> value?
compare_and_set(name: string, expected: value?, new_value: value) -> bool
list(prefix: string) -> [name]
```
Backs: Branches, the manifest's `default_branch` pointer.

### 2.4 `Capability.Storage.Index` (v1, optional/rebuildable)
Query surface over derived data. Unlike the three above, no Provider is
*required* to implement this efficiently — a Provider MAY answer Index
queries by falling back to a full scan, since Index content is always
rebuildable from the other three contracts.
```
query(spec: QuerySpec) -> stream<result>
rebuild() -> void
```

## 3. Why four contracts and not one unified "storage" interface

A single unified interface would either (a) expose least-common-denominator
operations (defeating the purpose of specialized backends like SQLite's
transactions or an object store's cheap immutable blobs), or (b) leak
storage-specific concepts upward (violating the technology-neutral tenet).
Four small contracts, each matching one genuinely different access pattern
(immutable content-addressed, append-only ordered, mutable pointer with
CAS, best-effort query), let each Provider implement only what it's good
at and let the Repository core compose them without knowing which
technology answers which call.

## 4. Provider matrix (illustrative, not exhaustive)

| Provider | BlobStore | AppendLog | RefStore | Index | Notes |
|---|---|---|---|---|---|
| Filesystem (reference) | content-addressed file tree, sharded by hash prefix | append-only files with fsync | lock file + atomic rename for CAS | flat-file scan or embedded SQLite for speed | Simplest to reason about; reference implementation for conformance testing |
| SQLite | BLOB column keyed by hash | table with monotonic rowid | table with `UPDATE ... WHERE value = expected` for CAS | native SQL queries | Best single-file embedded option; strong local ACID |
| PostgreSQL | BYTEA/large object keyed by hash | sequence-backed table | row with `SELECT ... FOR UPDATE` or native CAS | full SQL + extensions (e.g. full-text, graph queries) | Best for multi-writer server deployments; explicitly never a hard dependency |
| Cloud Object Store (S3-like) + metadata DB | native object PUT/GET keyed by hash | requires an auxiliary ordering service (object stores have no native strict order) | requires a small strongly-consistent side-store (e.g. DynamoDB conditional write) | delegated to a search service | Illustrates that AppendLog/RefStore are the hard contracts for "storage-less" backends — called out explicitly so implementers don't assume every Provider is trivial |
| In-Memory | map keyed by hash | Vec/list | map with CAS via mutex | linear scan | Used for testing; discarded on process exit |

## 5. Provider negotiation

A Repository's manifest ([01](01-repository-specification.md) §3) records
which Provider backs each Capability via `provider_bindings`. Nothing
prevents mixing — e.g. Filesystem BlobStore + SQLite RefStore — as long as
each bound Provider correctly implements its contract's semantics
(atomicity of CAS, durability of append, byte-fidelity of content
addressing). Conformance testing (out of scope here, tracked as a Phase 2
deliverable) verifies a Provider satisfies a contract via a
technology-agnostic test suite exercised against the contract interface,
never against a specific Provider's internals.

## 6. Explicit non-goal

The Repository never queries a Provider for its native query language
(SQL, S3 Select, etc.) from core logic — that would leak the technology
choice into behavior other Providers couldn't replicate. `Capability.Storage.Index`
exists precisely to let a Provider *optionally* expose a rich query
surface without the Repository core depending on it being present or
consistent in shape across Providers.

## 7. Trade-off: why not just standardize on SQLite everywhere

**Alternative considered:** require every Provider to be SQLite-compatible
(simplifies Milestone 1, gives free ACID transactions and free single-file
packaging). Rejected as the *architecture's* baseline — it would violate
"the repository shall not depend upon PostgreSQL" generalized to "shall not
depend on any specific engine," and would make it structurally harder to
add a genuinely storage-less Provider (e.g. a pure cloud object store)
later, since application code would have implicitly grown to assume SQL
semantics. SQLite remains an excellent *default local Provider* (see
[09-cli-specification.md](09-cli-specification.md) Milestone 1 choice) —
the rejection is of SQLite as an architectural requirement, not as an
implementation choice. See
[ADR-0006](../adr/ADR-0006-provider-capability-contracts.md).
