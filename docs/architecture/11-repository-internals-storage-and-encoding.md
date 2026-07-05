# Repository Internals I — Physical Layout, Encoding, and Data Movement

Status: Draft v0.1 (Sprint 0.5) · Architecture only — no production implementation is specified here. This document exists to give implementers enough architectural constraint to build a conformant Provider without inventing physical-layer decisions ad hoc.

This document covers the physical/mechanical layer beneath the logical
model in [02-object-storage-architecture.md](02-object-storage-architecture.md)
and [06-storage-provider-abstraction.md](06-storage-provider-abstraction.md).
Nothing here changes the logical model; it constrains how Providers may
realize it.

## 1. Physical repository layout (reference Filesystem Provider)

The logical layout in
[01-repository-specification.md](01-repository-specification.md) §2 maps
to physical directories as follows in the reference Provider:

```
<root>/
  manifest.json
  config/
  objects/<h0><h1>/<full-hash>              # sharded 2-level by hash prefix
  relationships/<h0><h1>/<full-hash>        # relationships are Objects (02 §5); same physical scheme
  events/segment-<n>.log
  commits/<h0><h1>/<full-hash>
  branches/<name>.ref
  journal/segment-<n>.jrnl
  index/...                                  # Provider-specific, always rebuildable
  attachments/<h0><h1>/<full-hash>
  integrity/merkle-roots.json
  workspace/...                               # never packaged, never synced
```

Two-level hash-prefix sharding (`<h0><h1>` = first two hex bytes of the
hash) keeps any single directory's entry count bounded even at millions of
objects, avoiding filesystem-specific directory-size pathologies without
requiring any Provider-specific configuration.

## 2. Object serialization

Every Engineering Object revision's payload is serialized to bytes via a
**canonical serialization** before hashing or storage. Canonical means: a
fixed key ordering (lexicographic), a fixed number encoding (no
locale-dependent or platform-dependent float formatting), and no
insignificant whitespace — the same logical content must always produce
the same bytes, on every platform, in every conformant implementation.
This is the property [01-repository-specification.md](01-repository-specification.md)
§5 depends on for cross-implementation conformance.

**Recommendation:** a deterministic subset of CBOR (Concise Binary Object
Representation, RFC 8949) for canonical serialization of structured
payloads, with the RFC's own "deterministic encoding" rules (§4.2)
adopted directly rather than re-invented. Rationale: CBOR is a compact,
IETF-standardized binary format with an existing deterministic-encoding
specification, avoiding the trap of OEP inventing its own canonicalization
rules (the same standardization argument used for UUIDv7 in
[ADR-0002](../adr/ADR-0002-object-identity.md)). JSON remains an
acceptable **display/interchange** representation (e.g. `oep object show`
output, the manifest) but is never the canonical hashing representation,
since JSON has no single canonical byte-level form across implementations.

## 3. Object packing

Individual small objects stored as individual files (or blob rows) incurs
overhead at scale (filesystem inode overhead, seek cost). Providers MAY
pack multiple immutable, content-addressed objects into a single **pack
file** — a simple concatenation of `{content_hash, length, bytes}` records
plus an index mapping hash → offset within the pack. This is purely a
Provider-level optimization: from the `BlobStore` Capability contract's
perspective ([06](06-storage-provider-abstraction.md) §2.1), packing is
invisible — `get(hash)` and `put(content)` behave identically whether the
Provider stores one object per file or many objects per pack.

Packing is **not required** for conformance; it is a performance strategy
a Provider may adopt independently, and different Providers may pack
differently (or not at all) without affecting cross-implementation
determinism, since packing never changes content hashes.

## 4. Compression strategy

Compression is a Provider/pack-level concern, never part of the canonical
serialization (compressing before hashing would make the hash
implementation- and setting-dependent, breaking determinism). The
recommended layering:

1. Canonically serialize the payload → bytes.
2. Compute `content_hash` over those exact bytes.
3. A Provider MAY compress the bytes for physical storage (e.g. Zstd),
   recording the codec alongside the stored blob, and MUST decompress back
   to the exact original bytes on `get` before the hash is ever re-verified
   against them.

This guarantees `content_hash` is always a hash of canonical, uncompressed
content — compression is purely a storage-density optimization that must
be perfectly reversible and invisible above the `BlobStore` boundary.

## 5. Streaming support

Large payloads (CAD files, firmware images, bulk sensor captures) must not
force an implementation to hold the full byte content in memory. The
`BlobStore` Capability ([06](06-storage-provider-abstraction.md) §2.1) is
specified with a streaming-friendly extension in mind:

```
put_stream(content: byte_stream, expected_length?: uint64) -> ContentHash
get_stream(hash: ContentHash) -> byte_stream
```

Hashing is computed incrementally as bytes stream through (BLAKE3 supports
incremental/streaming hashing natively, part of why it was selected in
[ADR-0008](../adr/ADR-0008-hash-algorithm-agility.md)), so `put_stream`
never requires buffering the whole payload to compute its address.

## 6. Memory mapping

Providers operating over local files MAY use memory-mapped I/O (mmap) for
read access to large content-addressed blobs, avoiding a full-copy read
into process memory for random-access consumers (e.g. an Engineering
Studio viewer paging through a large attachment). This is purely a local
Filesystem Provider optimization detail — the `BlobStore` contract's
`get`/`get_stream` calls are agnostic to whether the Provider serves bytes
from an mmap'd region or a network read; no other layer may assume mmap is
available, since remote/cloud Providers cannot offer it.

## 7. Large object strategy

Objects above a size threshold (recommendation: 4 MiB, tunable per
deployment, not a normative constant) are treated as **Attachments**
rather than inline revision payloads:

- The Engineering Object's revision payload stores an `attachment_ref`
  (the content hash of the large blob) rather than the blob itself.
- The Attachment is stored, addressed, and streamed exactly like any other
  content-addressed blob ([02](02-object-storage-architecture.md) §4), just
  through the `attachments/` area to let Providers apply different
  physical handling (e.g. different pack-file thresholds, different
  compression defaults tuned for large binaries vs. small structured data).
- This split exists purely for physical efficiency (small metadata objects
  stay fast to enumerate/index; large binaries don't bloat every
  object-listing scan) — logically, an Attachment reference is still just
  a Content Hash, no new identity concept.

## 8. Lazy loading

`oep checkout`/reconstruction ([09-cli-specification.md](09-cli-specification.md))
MUST NOT require materializing every Attachment's full bytes to reconstruct
repository *state* — an Engineering Object's current revision and its
relationships are resolvable from the Object Store and Relationship Index
without touching Attachment content. Attachment bytes are fetched lazily,
on first access to that specific attachment reference, exactly mirroring
how a Git checkout resolves tree/commit metadata without necessarily
touching every blob's content until read.

## 9. Reference resolution

Cross-object references (an `attachment_ref`, a Relationship's `subject`/
`object` fields, a Commit's `object_changes` entries) are always Content
Hashes or EOID/Revision ID pairs — never physical addresses (file paths,
row IDs, memory offsets). Resolution is always: take the logical
reference, ask the bound `BlobStore`/`RefStore` Capability to resolve it.
This indirection is what makes packing (§3), compression (§4), and
compaction ([12](12-repository-internals-lifecycle-and-maintenance.md))
possible without ever invalidating a stored reference — physical location
can always change; logical references never need to.
