# Block Layout — `.oep` Block Types v0.1

Status: Draft v0.1 (Sprint 0.5) · Defines every Block type in the native container ([16-native-container-format.md](16-native-container-format.md)).

## 1. Common Block framing

Every Block, regardless of type, shares one framing structure:

```
BlockHeader (fixed 64 bytes, itself 64-byte aligned) {
  block_magic      : 4 bytes    // "BLK1"
  block_type       : 2 bytes    // enum, see §2
  flags            : 2 bytes    // bit 0 = compressed (§4), bits 1-15 reserved
  payload_length   : 8 bytes    // length of payload, excluding this header and padding
  content_hash     : 32 bytes   // hash of the *uncompressed* canonical payload
  codec            : 2 bytes    // 0 = none, 1 = Zstd (only meaningful if flags bit0 set)
  reserved         : 14 bytes   // zero-filled in v0.1
}
Payload (payload_length bytes, then zero-padded to next 64-byte boundary)
```

`content_hash` is computed exactly as specified in
[02-object-storage-architecture.md](02-object-storage-architecture.md) §4
and [11-repository-internals-storage-and-encoding.md](11-repository-internals-storage-and-encoding.md)
§4 — over canonical, uncompressed bytes — so a Block's content hash is
identical whether or not the container chose to compress it physically,
preserving determinism.

## 2. Block type registry

| `block_type` | Name | Carries |
|---|---|---|
| `0x0001` | Manifest Block | Repository manifest ([01-repository-specification.md](01-repository-specification.md) §3) |
| `0x0002` | Object Block | One Engineering Object revision payload ([02-object-storage-architecture.md](02-object-storage-architecture.md)) |
| `0x0003` | Relationship Block | One Relationship revision (a Relationship is an Object, [02](02-object-storage-architecture.md) §5, but is given its own type code to let readers filter the Block Stream for graph-only traversal without touching non-relationship Object Blocks) |
| `0x0004` | Event Block | One Event record ([03-commit-architecture.md](03-commit-architecture.md) §4) |
| `0x0005` | Commit Block | One Commit record ([03-commit-architecture.md](03-commit-architecture.md) §2) |
| `0x0006` | Branch Block | One Branch record ([04-branch-architecture.md](04-branch-architecture.md) §1) |
| `0x0007` | Attachment Block | One large-object payload ([11-repository-internals-storage-and-encoding.md](11-repository-internals-storage-and-encoding.md) §7); MAY span multiple physical Blocks via the chunking convention in §5 |
| `0x0008` | Index Block | One fragment of the Container Index ([18-container-index.md](18-container-index.md)) — present in the Block Stream only for the append-friendly incremental case (§ [16](16-native-container-format.md) §8); the authoritative, current Index location is always the one the Header/Footer point to |
| `0x0009` | Signature Block | A detached signature over a `content_hash` (commit signature, package-level signature — [08-integrity-verification-strategy.md](08-integrity-verification-strategy.md) §3) |
| `0x000A` | Extension Block | Forward-compatible, self-describing payload for future Block kinds not yet standardized — see §10 |

`0x000B`–`0xFFFF` reserved for future standardized types (minor version
additions); `0xFF00`–`0xFFFE` reserved for private/experimental use by
implementations, never to be relied upon for interoperability.

## 3. Why Relationship gets its own Block type despite being an Object

[ADR-0003](../adr/ADR-0003-relationships-as-objects.md) established that a
Relationship is logically an Engineering Object — that decision is
unchanged. Giving Relationships a distinct `block_type` at the *physical
container* layer is purely a streaming/filtering optimization: a reader
building only the Relationship Index ([18-container-index.md](18-container-index.md))
can skip non-relationship Object Blocks by type code alone, without
deserializing their payload to check a `type` field. This does not
reintroduce a second identity scheme (the constitutional concern behind
ADR-0003) — a Relationship Block's payload is still exactly an Engineering
Object revision with `type = "Relationship""`; the distinct Block type is
a container-format-level indexing hint, not a logical-model change.

## 4. Compression

Per-Block compression is optional (`flags` bit 0) and Block-type-agnostic.
Recommendation: Zstd for Object/Relationship/Event/Commit Blocks
(structured, often-repetitive content benefits from a general-purpose
dictionary-capable codec); Attachment Blocks default to uncompressed
(`codec = 0`) since large binary engineering artifacts (CAD, firmware,
sensor captures) are frequently already compressed at the source format
level, and forcing a second compression pass wastes CPU for negligible
gain. A container MAY mix compressed and uncompressed Blocks freely;
`content_hash` is unaffected either way (§1).

## 5. Attachment chunking

An Attachment exceeding a configurable chunk threshold (recommendation:
16 MiB per chunk) is split across multiple sequential Attachment Blocks
sharing one logical `content_hash` (the hash of the full reassembled
payload) via a chunk-header convention:

```
AttachmentChunkHeader (first 32 bytes of an Attachment Block's payload) {
  full_content_hash : 32 bytes   // identifies the complete Attachment across all chunks
  chunk_index       : 4 bytes
  chunk_count       : 4 bytes
}
```

This lets a reader stream an Attachment progressively (§ [19-streaming-model.md](19-streaming-model.md))
without holding the entire large object in memory, while still exposing
one coherent Content Hash to the logical model above it.

## 6–9. Manifest, Commit, Branch, Signature Block payloads

These four Block types carry payloads that are already fully specified at
the logical layer and are not redefined here — a Manifest Block's payload
is exactly the manifest structure from
[01-repository-specification.md](01-repository-specification.md) §3, a
Commit Block's payload is exactly the Commit structure from
[03-commit-architecture.md](03-commit-architecture.md) §2, a Branch
Block's payload is exactly the Branch structure from
[04-branch-architecture.md](04-branch-architecture.md) §1, and a
Signature Block's payload is a detached signature structure per
[08-integrity-verification-strategy.md](08-integrity-verification-strategy.md)
§3, each canonically serialized per
[11-repository-internals-storage-and-encoding.md](11-repository-internals-storage-and-encoding.md)
§2 before being written as a Block payload. The Block layer never
redefines logical structure — it only frames, hashes, and optionally
compresses it.

## 10. Extension Blocks

An Extension Block's payload begins with a self-describing header so an
unaware reader can decide to skip it (per
[16-native-container-format.md](16-native-container-format.md) §7)
without any registry lookup:

```
ExtensionHeader (first 48 bytes of payload) {
  extension_id   : 32 bytes   // hash of a reverse-DNS namespaced string, e.g. "oep.sim.ResultSet" — avoids a central registry bottleneck
  extension_ver  : 2 bytes
  reserved       : 14 bytes
}
```

A reader that does not recognize `extension_id` treats the entire Block
as opaque and safely ignores it — it still contributes to Index-level
integrity verification ([18-container-index.md](18-container-index.md)
§4) even when its content is not interpreted, so a package remains
verifiably intact end-to-end regardless of which optional extensions a
given reader understands.
