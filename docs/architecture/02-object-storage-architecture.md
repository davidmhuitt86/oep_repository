# Object Storage Architecture

Status: Draft v0.2 (RC2) · Defines Engineering Object identity, revisions, content addressing, and the Relationship model built on top of it.

This document specifies the **Engineering Object Store** layer — see
[10-layered-architecture.md](10-layered-architecture.md) for how it
relates to the Repository and Package layers above it.

## 1. Three independent identities

RC2 makes explicit what RC1 left implicit: an Engineering Object carries
**three separate identities**, each with its own purpose and lifecycle.
None is derived from another, and in particular **identity is never
derived solely from content hashing** — see
[ADR-0010](../adr/ADR-0010-three-part-identity.md).

| Identity | Mutability | Purpose | Derived from |
|---|---|---|---|
| **Engineering Object ID (EOID)** | Permanent | Names the engineering *concept* itself, independent of any state it has ever been in. | Nothing — minted once, opaque forever. |
| **Revision ID** | Append-only, one new value per change | Names *a specific revision* in a human-usable, revision-oriented way; supports walking revision history independently of whether content ever repeats. | A per-object ordering scheme (§3), not content. |
| **Content Hash** | Immutable, derived | Names the *bytes* of one revision's canonical serialization; used for integrity, deduplication, and synchronization. | The canonical serialization of the revision payload. |

Why three, not two: collapsing Revision ID into Content Hash (RC1's
approach — `RevisionID = Hash(...)`) makes two revisions with identical
content indistinguishable as revisions, which breaks "revision N of this
object" as a concept independent of whether the bytes happen to match
another revision (e.g. reverting a component to a prior configuration is
revision N+1, not "the same revision as N-3," even though the content is
byte-identical). Keeping Revision ID and Content Hash independent answers
"which revision is this" and "is this content identical to some other
content" with the identity built for each question.

An Engineering Object is therefore not "a file that changes" — it is a
permanent identity (EOID) with a growing, ordered sequence of Revisions,
each independently named (Revision ID) and independently content-addressed
(Content Hash), mirroring how the platform treats engineering history in
general (nothing silently modified; corrections produce new events).

## 2. Engineering Object ID (EOID) design

**Recommendation: UUIDv7.** Trade-off analysis:

| Option | Pros | Cons | Verdict |
|---|---|---|---|
| Sequential integer | Simple, compact | Requires central allocation, breaks offline-first and merge (two branches created offline would collide) | Rejected |
| UUIDv4 | No central allocation, offline-safe | Not time-sortable, harms index locality at scale | Rejected |
| ULID | Time-sortable, offline-safe | Non-standard outside a narrow community, weaker long-term standardization prospects for a 20-year format | Rejected |
| **UUIDv7** | Time-sortable, offline-safe, IETF-standardized (RFC 9562), no central allocation | Slightly larger than a sequential int | **Selected** |

See [ADR-0002](../adr/ADR-0002-object-identity.md). UUIDv7 satisfies
offline-first (any node can mint an EOID without coordination), determinism
requirements (two implementations generating IDs independently never
collide), and long-term standardization (it is an IETF RFC, not a
project-specific scheme) simultaneously.

## 3. Revision ID design

A Revision ID is scoped to its EOID (revision numbering is per-object, not
global) and is **sequential by default**:

```
RevisionID(EOID, n) = <EOID>.r<n>          // e.g. 018f...c2.r7 — the 8th revision
```

`n` is a monotonically increasing integer starting at 0, assigned when a
revision is recorded. This is deliberately human-usable ("revision 7 of
this component") and independent of content — two revisions of the same
object never collide in numbering even if their content is identical or
their commits happen concurrently on different branches, because
numbering is a property of the object's revision *sequence*, not of
content or of wall-clock time.

**Alternative naming schemes considered:** a hash-of-content revision
identifier (RC1's original design, rejected — see §1); a globally unique
revision UUID with no ordering semantics (rejected — loses the
human-usable "which came after which" property without gaining anything,
since ordering is already recoverable from `previous_revision_id`); a
timestamp-based revision identifier (rejected — clock skew across offline
nodes makes strict ordering unreliable, exactly the problem EOID's UUIDv7
choice already had to solve once). Sequential-per-object integer suffixed
onto the permanent EOID is simplest, is stable under merge (a merge simply
allocates the next integer for the reconciled branch, per
[04-branch-architecture.md](04-branch-architecture.md) §3), and needs no
coordination beyond the single object's own history, which is already
serialized by the Commit that introduces each revision.

Each Revision record separately stores:

| Field | Description |
|---|---|
| `eoid` | Permanent identity this revision belongs to |
| `revision_id` | Sequential, human-usable identity of this specific revision (this section) |
| `content_hash` | Hash of the canonical serialization of this revision's payload — independent of `revision_id` (§1, §4) |
| `previous_revision_id` | The Revision ID this one supersedes (absent for revision 0) |
| `commit_id` | The Commit (see [03](03-commit-architecture.md)) that introduced this revision — ties every revision to the exact engineering-history moment it was created |

Tampering detection no longer relies on Revision ID chaining alone (since
Revision ID is not content-derived): it relies on `content_hash` mismatches
(content-level tampering) combined with the Commit DAG's own hash chain
(history-level tampering, [08](08-integrity-verification-strategy.md)) —
Revision ID's role is purely addressing and ordering, not integrity.

An Engineering Object record (independent of any specific revision) stores:

| Field | Description |
|---|---|
| `eoid` | Permanent identity |
| `type` | Engineering Object type (Component, Document, Measurement, Rule, ...) — extensible, not a closed enum |
| `current_revision_id` | Revision ID of the latest revision on the object's home branch |
| `created_at`, `created_by` | Provenance of the identity, not of any particular revision |

## 4. Content storage

Revision payloads are stored content-addressed: the storage key for a
revision's content is `content_hash` itself. This gives, for free:

- **Deduplication** — identical content (e.g. the same firmware binary
  attached to two components) is stored once regardless of how many
  Engineering Objects reference it.
- **Verifiability** — the content hash is the address; any Provider that
  returns bytes that don't hash to the requested address is provably
  corrupt or malicious.
- **Provider independence** — a content-addressed get/put contract is the
  smallest possible Storage Capability surface, easy for any Provider
  (filesystem, SQLite blob column, S3 object, in-memory map) to implement.

Large binary payloads (CAD files, firmware images, captured sensor data)
are stored in the same content-addressed scheme but through the
`attachments` area, letting Providers apply different physical handling
(e.g. streaming instead of loading fully into memory) without changing the
addressing model.

## 5. Relationships

A Relationship is an Engineering Object (type = `Relationship`) whose
payload is:

```
{
  "relationship_type": "connects_to" | "part_of" | "measured_by" | ...,
  "subject": <EOID>,
  "object": <EOID>,
  "attributes": { ... }         // typed, relationship-type-specific
}
```

Because a Relationship is an Engineering Object, it automatically gets
identity, revision history, and can itself be the subject of further
relationships (e.g. "this connection was validated by that Rule") with no
additional storage subsystem. The **Relationship Index** is a derived,
rebuildable structure (adjacency lists keyed by subject/object/type) that
makes graph traversal efficient; it lives under `index/`
([01](01-repository-specification.md) §2) and is never a source of truth.

**Alternative considered:** a dedicated relationship store with its own ID
space, separate from the Object Store. Rejected because it introduces a
second identity scheme, violates the closed-primitive-set tenet
([00](00-repository-architecture.md) §3), and gains nothing a derived
index doesn't already provide — see
[ADR-0003](../adr/ADR-0003-relationships-as-objects.md).

## 6. Object types are open, not closed

`type` is an extensible string namespace (recommendation:
reverse-DNS-style, e.g. `oep.core.Component`, `acme.custom.PressureSensor`),
never a fixed enum baked into the Repository. The Repository does not
validate type-specific schema — that is a Capability provided by higher
layers (validation rules, Canon). The Repository's only obligations to
`type` are: store it, index it, and never interpret its payload structure
beyond canonical serialization for hashing.
