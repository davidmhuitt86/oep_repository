# Native `.oep` Container Format — Physical Specification v0.1

Status: Draft v0.1 (Sprint 0.5) · Normative for byte layout. Supersedes ZIP-based packaging per [ADR-0014](../adr/ADR-0014-native-container-format.md).

This document specifies the **Engineering Package** layer
([10-layered-architecture.md](10-layered-architecture.md)) as a native
binary container. It defines byte layout only; block *content* semantics
are in [17-block-layout.md](17-block-layout.md), the index in
[18-container-index.md](18-container-index.md), streaming behavior in
[19-streaming-model.md](19-streaming-model.md), and maintenance
operations in [20-package-maintenance.md](20-package-maintenance.md).

## 1. Design principles

1. **Header-forward.** Everything a reader needs to validate the file and
   locate the index is in the first fixed-size region — no requirement to
   seek to the end of the file before doing anything useful, unlike a
   ZIP central directory. This is the direct fix for the streaming
   weakness identified in [ADR-0013](../adr/ADR-0013-container-format-reevaluation.md).
2. **Block-structured.** The file is a sequence of self-describing,
   independently-verifiable Blocks. A Block is the unit of content
   addressing, streaming, and random access.
3. **Append-friendly.** New Blocks can be appended without rewriting
   existing ones, and a trailing Footer can be rewritten cheaply to point
   at new Blocks — supporting incremental updates (§8) without full
   repacking.
4. **Self-verifying at every layer.** Every Block carries its own
   integrity hash; the Index carries a hash over Block hashes; the Footer
   carries a hash over the Index. Corruption anywhere is detectable
   without trusting any single region.
5. **No silent version guessing.** A reader that doesn't recognize the
   major format version refuses to open the file rather than
   misinterpret it, matching
   [01-repository-specification.md](01-repository-specification.md) §4.2.

## 2. Endianness and alignment

- All multi-byte integers are **little-endian**. Rationale: little-endian
  is the native byte order of the overwhelming majority of deployed CPUs
  (x86-64, ARM in its default mode), avoiding a byte-swap on the hot path
  for nearly all readers; the format declares this explicitly rather than
  leaving it to host-order (which would make the format non-portable) or
  network-order/big-endian (which would tax the common case for a
  standard-that-declares-explicitly benefit that content hashing already
  provides via the header's own magic-number check).
- All Blocks are aligned to **64-byte boundaries**. Rationale: 64 bytes
  covers the common cache-line size (64 bytes on nearly all contemporary
  x86-64 and ARM parts) and is a safe common denominator for direct I/O
  and memory-mapped access alignment requirements across platforms,
  without over-fitting to any one platform's exact page or cache-line
  size. Padding bytes between Blocks are zero-filled and excluded from
  content hashing (they are framing, not content).

## 3. File layout

```
┌────────────────────────────────────────────────────────────┐
│ Header (fixed 256 bytes)                                    │  offset 0
├────────────────────────────────────────────────────────────┤
│ Reserved Region (fixed 256 bytes, zero-filled in v0.1)       │  offset 256
├────────────────────────────────────────────────────────────┤
│ Block Stream (variable length, 64-byte aligned)              │  offset 512
│   Block 0 │ Block 1 │ Block 2 │ ... │ Block N               │
├────────────────────────────────────────────────────────────┤
│ Container Index (variable length)                            │  see 18-container-index.md
├────────────────────────────────────────────────────────────┤
│ Footer (fixed 128 bytes)                                     │  last 128 bytes of file
└────────────────────────────────────────────────────────────┘
```

## 4. Header (offset 0, 256 bytes, fixed)

| Field | Size | Description |
|---|---|---|
| `magic` | 8 bytes | ASCII `"OEPKG1\0\0"` — identifies the file type and format family at a glance (visible even in a hex dump or `file`-utility magic database entry) |
| `format_major` | 2 bytes | Major version. Readers MUST refuse to open a file with an unrecognized major version. |
| `format_minor` | 2 bytes | Minor version. Readers MUST accept unrecognized minor versions in read-only mode, ignoring unrecognized optional structures (§7). |
| `hash_algorithm` | 1 byte | Enum: `0x01 = BLAKE3`, `0x02 = SHA-256` (mandatory fallback per [ADR-0008](../adr/ADR-0008-hash-algorithm-agility.md)) |
| `repository_id` | 16 bytes | UUIDv7 of the source Repository ([01-repository-specification.md](01-repository-specification.md) §1) |
| `created_at` | 8 bytes | Unix milliseconds, creation time of this package |
| `package_kind` | 1 byte | Enum: `0x01 = full`, `0x02 = shallow`, `0x03 = delta` (semantics per [07-package-format-proposal.md](07-package-format-proposal.md) §5, retained conceptually even though that document's container mechanics are superseded) |
| `flags` | 4 bytes | Bitfield: bit 0 = signed, bit 1 = compressed blocks present, bits 2–31 reserved (must be zero in v0.1) |
| `index_offset` | 8 bytes | Absolute byte offset of the Container Index |
| `index_length` | 8 bytes | Byte length of the Container Index |
| `header_hash` | 32 bytes | Hash (per `hash_algorithm`) of all preceding Header bytes; lets a reader detect Header corruption before trusting any other field |
| *padding* | remainder to 256 | Zero-filled |

Putting `index_offset`/`index_length` in the Header — not only in the
Footer — is what makes the format genuinely header-forward: a reader with
random access can jump straight to the Index without reading the Footer
at all; the Footer (§6) exists for the pure sequential-streaming case
where the Index location isn't known until the stream is fully consumed.

## 5. Reserved Region (offset 256, 256 bytes, fixed)

Zero-filled in v0.1. Reserved for future mandatory structures that must be
readable without a minor-version-aware reader understanding them (e.g. a
future mandatory capability-negotiation record). A v0.1 reader MUST
verify this region is all-zero when `format_major` is 1; a future major
version may define its contents, at which point v0.1 readers will already
have refused to open the file via the major-version check (§4).

## 6. Footer (last 128 bytes, fixed)

| Field | Size | Description |
|---|---|---|
| `index_offset` | 8 bytes | Duplicated from Header, for pure-sequential readers who reach EOF before knowing where the Index started |
| `index_length` | 8 bytes | Duplicated from Header |
| `index_hash` | 32 bytes | Hash of the full Container Index — the top of the format's integrity chain (Block → Index → Footer), see [18-container-index.md](18-container-index.md) §4 |
| `block_count` | 8 bytes | Total number of Blocks in the Block Stream |
| `footer_magic` | 8 bytes | ASCII `"OEPEND1\0"` — lets a reader confirm the file was not truncated |
| *padding* | remainder to 128 | Zero-filled |

A reader doing a pure forward stream (§ [19-streaming-model.md](19-streaming-model.md))
can process every Block as it arrives without ever needing the Footer;
the Footer exists to let a *sequential-only* reader (e.g. reading from a
non-seekable pipe) discover where the Index would have been, for
after-the-fact validation once the full stream is buffered, and to let any
reader confirm the file wasn't truncated (`footer_magic` present at the
expected trailing offset).

## 7. Version negotiation

- **Major version bump**: incompatible change to Header layout, Block
  framing, or hashing rules. Old readers MUST refuse; this is the only
  case requiring a hard compatibility break.
- **Minor version bump**: new optional Block types, new optional Header
  flags, new optional Index structures. Old readers MUST be able to open
  the file, read every Block type they recognize, and safely skip Block
  types they don't (§ [17-block-layout.md](17-block-layout.md) §1, every
  Block declares its own length so skipping is always possible without
  understanding the content).
- A reader reports the highest format version it understands via its own
  Capability advertisement ([22-repository-capability-model.md](22-repository-capability-model.md)),
  so tooling can detect "this reader may not fully understand this
  package" without guessing from the file alone.

## 8. Incremental updates and append-friendliness

Adding new content to an existing `.oep` file does not require rewriting
existing Blocks:

1. New Blocks are appended after the current last Block (at the next
   64-byte-aligned offset).
2. A new Container Index is written after the new Blocks (superseding the
   old Index's location).
3. The Footer (and Header's `index_offset`/`index_length`) are rewritten
   to point at the new Index.

This means an incremental update's I/O cost is proportional to the new
content plus one Index rewrite, not the whole file — directly serving the
"incremental updates" and "large engineering repositories" design goals.
The old Index bytes become dead space (reclaimed during
[20-package-maintenance.md](20-package-maintenance.md) repacking), never
silently reinterpreted, since a stale Index is only reachable if something
still points `index_offset` at it, which the atomic Header/Footer rewrite
in step 3 prevents.

## 9. Future extension strategy

Two escape hatches, defined now specifically so later versions don't need
to break the format to add capability:

1. **Extension Blocks** ([17-block-layout.md](17-block-layout.md) §10) —
   a Block type reserved for forward-compatible, self-describing
   extensions; old readers skip them safely per §7.
2. **Reserved Region** (§5) — for the rare case where a future mandatory
   (non-skippable) structural addition is needed; consuming it requires a
   major version bump by definition, kept deliberately rare.
