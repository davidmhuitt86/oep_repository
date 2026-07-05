# Object Storage Architecture

Status: Draft v0.1 · Defines Engineering Object identity, revisions, content addressing, and the Relationship model built on top of it.

## 1. Identity vs. content

Per the constitutional principle, identity and content are strictly
separated:

| Concept | Mutability | Description |
|---|---|---|
| **Object ID (OID)** | Permanent | Assigned once, at creation, never reused, never changes even if content is completely replaced. |
| **Revision** | Append-only | Each change to an object's content produces a new Revision; prior revisions remain readable forever. |
| **Content Hash** | Derived | A hash of the canonically serialized content of one revision. |

An Engineering Object is therefore not "a file that changes" — it is a
permanent identity with a growing, immutable chain of revisions, exactly
mirroring how the platform treats engineering history in general (nothing
silently modified; corrections produce new events).

## 2. Object ID design

**Recommendation: UUIDv7.** Trade-off analysis:

| Option | Pros | Cons | Verdict |
|---|---|---|---|
| Sequential integer | Simple, compact | Requires central allocation, breaks offline-first and merge (two branches created offline would collide) | Rejected |
| UUIDv4 | No central allocation, offline-safe | Not time-sortable, harms index locality at scale | Rejected |
| ULID | Time-sortable, offline-safe | Non-standard outside a narrow community, weaker long-term standardization prospects for a 20-year format | Rejected |
| **UUIDv7** | Time-sortable, offline-safe, IETF-standardized (RFC 9562), no central allocation | Slightly larger than a sequential int | **Selected** |

See [ADR-0002](../adr/ADR-0002-object-identity.md). UUIDv7 satisfies
offline-first (any node can mint an OID without coordination), determinism
requirements (two implementations generating IDs independently never
collide), and long-term standardization (it is an IETF RFC, not a
project-specific scheme) simultaneously.

## 3. Revision chain

Each revision is content-addressed and cryptographically chained:

```
RevisionID = Hash( OID || revision_index || content_hash || previous_revision_id || commit_id )
```

- `content_hash` = hash of the canonical serialization of this revision's
  payload (the actual typed engineering content — e.g. a component's
  attributes).
- `previous_revision_id` = the RevisionID this one supersedes (absent for
  revision 0).
- `commit_id` = the Commit (see [03](03-commit-architecture.md)) that
  introduced this revision. This ties every revision to the exact
  engineering-history moment it was created, not just to its predecessor.

This makes a revision chain a per-object hash chain nested inside the
global commit DAG — tampering with any past revision is detectable both
locally (chain breaks) and globally (commit hash changes,
[08](08-integrity-verification-strategy.md)).

An Engineering Object record (independent of the revision chain) stores:

| Field | Description |
|---|---|
| `oid` | Permanent identity |
| `type` | Engineering Object type (Component, Document, Measurement, Rule, ...) — extensible, not a closed enum |
| `current_revision` | RevisionID of the latest revision on the object's home branch |
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
  "subject": <OID>,
  "object": <OID>,
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
