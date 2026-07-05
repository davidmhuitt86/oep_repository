# Container Index — `.oep` v0.1

Status: Draft v0.1 (Sprint 0.5) · Specifies random access, lookup, and integrity verification over the Block Stream ([17-block-layout.md](17-block-layout.md)).

## 1. Purpose

The Block Stream ([16-native-container-format.md](16-native-container-format.md)
§3) is a flat sequence with no inherent lookup structure — finding "the
Block for Content Hash X" by scanning would be O(n) in package size. The
Container Index is the derived, authoritative-within-the-package lookup
structure that makes resolution O(1)-ish (hash table) or O(log n) (sorted
table), matching the same "index is a cache, not a second source of
truth" principle already established for the logical-model Index in
[01-repository-specification.md](01-repository-specification.md) §2 — a
corrupted or missing Index is always rebuildable by scanning the Block
Stream from scratch, since every Block is fully self-describing
([17-block-layout.md](17-block-layout.md) §1).

## 2. Index structure

```
ContainerIndex {
  entry_count   : 8 bytes
  entries[]     : IndexEntry     // sorted ascending by content_hash for binary search
  block_type_summary[] : { block_type: 2 bytes, count: 8 bytes, first_offset: 8 bytes }
                                  // lets a reader jump straight to "all Commit Blocks" etc.
}

IndexEntry (fixed 56 bytes) {
  content_hash   : 32 bytes   // primary lookup key
  block_offset   : 8 bytes    // absolute byte offset of the Block's BlockHeader
  block_length   : 8 bytes    // total bytes including header and padding, for a bounded read
  block_type     : 2 bytes    // duplicated from the Block itself, avoids a read just to filter by type
  reserved       : 6 bytes
}
```

Sorting `entries[]` by `content_hash` gives binary-search lookup with no
additional hash table needed in the file itself; an in-memory reader MAY
additionally build a hash map on load for O(1) average lookup — that's an
in-memory reader optimization, not a file-format requirement.

## 3. Fast object resolution

Resolving an EOID + Revision ID to bytes ([02-object-storage-architecture.md](02-object-storage-architecture.md)
§1-3) is a two-step lookup, not a Container Index responsibility on its
own: the Container Index only maps Content Hash → physical Block location.
The EOID/Revision → Content Hash mapping lives in the logical Object Store
records (themselves stored as Object Blocks), consistent with
[10-layered-architecture.md](10-layered-architecture.md)'s layering — the
container format resolves *content-addressed* lookups; it has no notion
of EOIDs, Revisions, Commits, or Branches as first-class index keys,
exactly as a Git packfile's own index resolves SHA hashes to offsets and
leaves ref/commit-graph semantics to a layer above it.

## 4. Integrity verification chain

The Container Index is the structural backbone of package-level integrity,
completing the chain that starts at individual Blocks:

```
Block.content_hash  (per Block, §17-block-layout.md §1)
        │  verified independently per Block on read
        ▼
Index integrity: hash(canonical(ContainerIndex bytes)) = Footer.index_hash
        │  one hash covers every entry, hence every Block's recorded
        │  location and declared hash
        ▼
Header.header_hash  covers the Header itself (16-native-container-format.md §4)
```

Verification levels map directly onto
[08-integrity-verification-strategy.md](08-integrity-verification-strategy.md)
§5:

| Level | Container-format check |
|---|---|
| 0 — Structural | Header magic/version valid; Footer present at expected offset; `index_hash` recomputes correctly over the Index bytes |
| 1 — Content | For each `IndexEntry`, read the referenced Block and recompute its `content_hash`; compare |
| 2 — History | Delegated to the logical Commit DAG walk ([08](08-integrity-verification-strategy.md) §5) — the container format's job stops at "every Block is present and matches its declared hash"; whether the *history* those Blocks encode is internally consistent is a logical-layer concern |

## 5. Index rebuild

If the Index is missing, truncated, or fails `index_hash` verification, a
reader MAY rebuild it by scanning the Block Stream sequentially from
offset 512 ([16-native-container-format.md](16-native-container-format.md)
§3), reading each `BlockHeader`, and reconstructing `entries[]` — this is
the container-format-level equivalent of `git index-pack`. Rebuild cost is
O(block count); it is expected to be an uncommon recovery path
([20-package-maintenance.md](20-package-maintenance.md) §4), not a normal
open-time operation, since Header-forward design (§ [16](16-native-container-format.md) §1)
means the Index location is known immediately in the non-corrupted case.

## 6. Block metadata beyond the Index

`block_type_summary[]` (§2) exists so common operations ("list all
branches," "walk all commits") don't require touching every `IndexEntry`
individually — they can jump to `first_offset` for a given `block_type`
and then walk the Block Stream forward, using each Block's own
`payload_length` (§17-block-layout.md §1) to skip to the next Block of
interest without consulting the full Index at all. This is a read-pattern
optimization layered on top of the primary content-hash lookup table, not
a separate integrity-relevant structure — its correctness is implied by,
and re-derivable from, `entries[]`.
