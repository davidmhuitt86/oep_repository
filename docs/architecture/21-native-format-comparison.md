# Native Format Comparison — Lessons Learned

Status: Draft v0.1 (Sprint 0.5) · Informative. Explains what `.oep` v0.1 ([16](16-native-container-format.md)–[20](20-package-maintenance.md)) adopted from, and deliberately rejected from, existing storage/container technologies.

The goal, per directive, is not to imitate any of the following — it is
to learn from each, adopt what serves a 50-year engineering-data format,
and reject what doesn't.

## ZIP

- **Advantages:** universal tooling, simple central-directory concept,
  per-entry compression choice, decades of stability.
- **Limitations:** central directory conventionally at file end forces a
  two-pass read for pure streaming; no native content-addressing (entries
  are path-keyed, not hash-keyed); no built-in cryptographic integrity
  beyond optional, rarely-used per-entry CRC-32 (weak, not
  cryptographic).
- **Lessons learned:** an index/directory structure separate from the
  content stream is a good idea (`.oep`'s Container Index, [18](18-container-index.md));
  per-entry/per-Block compression choice is worth keeping.
- **Adopted:** the general shape of "content region + directory + fixed
  footer."
- **Rejected:** directory-at-end-only placement (adopted header-forward
  instead, [16](16-native-container-format.md) §1); path-based addressing
  (adopted content-hash addressing instead, consistent with
  [ADR-0001](../adr/ADR-0001-content-addressable-storage.md)); weak
  per-entry checksums (adopted cryptographic hashing throughout instead).

## TAR

- **Advantages:** extremely simple sequential format, excellent for
  pure streaming (no directory at all — every entry is self-describing
  and readable in one pass), trivially appendable.
- **Limitations:** no random-access index at all (must scan the whole
  archive to find one entry), no compression built in (relies entirely on
  an external layer like gzip), no integrity verification of any kind.
- **Lessons learned:** TAR's "every entry is self-describing, sequential
  reading needs zero lookahead" property is exactly what
  [19-streaming-model.md](19-streaming-model.md) §1 wants — this is the
  strongest influence TAR had on `.oep`'s Block framing
  ([17-block-layout.md](17-block-layout.md) §1, every Block declares its
  own length).
- **Adopted:** self-describing sequential entries as the baseline
  streaming mode.
- **Rejected:** no index at all (`.oep` keeps TAR's streamability *and*
  adds an Index for the random-access case ZIP is good at and TAR is not
  — this dual-mode design is the central architectural synthesis of
  `.oep`); no built-in integrity (adopted mandatory per-Block hashing).

## SQLite

- **Advantages:** single-file portability, full ACID transactions, mature
  and extremely well-tested B-tree storage, powerful query language,
  proven multi-decade file-format stability commitment (SQLite explicitly
  promises long-term readability).
- **Limitations:** binary format not meaningfully diffable or streamable
  entry-by-entry; requires embedding a full relational database engine
  just to read a single file, a much heavier dependency than a Block
  parser; not content-addressed natively (rowid/B-tree-key addressed);
  page-based layout optimized for transactional mutation, not for
  content-addressed immutable data.
- **Lessons learned:** SQLite's "we will keep this format readable for
  decades, and back that promise with a permissively licensed reference
  implementation" strategy is exactly the credibility model
  [ADR-0014](../adr/ADR-0014-native-container-format.md) adopts for
  `.oep` — a single-project format can become a trusted long-term standard
  if the reference implementation itself is the trust anchor.
- **Adopted:** the "reference implementation as standardization strategy"
  philosophy; page/Block alignment discipline for I/O efficiency.
- **Rejected:** SQLite as the container itself (per
  [ADR-0007](../adr/ADR-0007-package-format.md), now further reinforced by
  [ADR-0014](../adr/ADR-0014-native-container-format.md) — `.oep`
  does not want to require every implementation to embed a relational
  engine just to read content-addressed Blocks); B-tree/rowid addressing
  (content hashing serves `.oep`'s dedup/verification/sync needs better).

## Git Packfiles

- **Advantages:** content-addressed by design (SHA-1/SHA-256 object
  hashes), append-friendly (new packs layer on top without rewriting old
  ones), a proven, decades-long track record for exactly this domain
  shape — a DAG of content-addressed, typed records (blobs, trees,
  commits) — which is structurally the closest existing analog to `.oep`'s
  own Object/Relationship/Commit/Branch model.
  Packfile-index (`.idx`) files give O(log n) offset lookup by hash,
  directly the model `.oep`'s Container Index borrows most heavily from.
- **Limitations:** packfile format is intentionally Git-internal and
  under-specified for external reuse (delta-compression scheme is
  intricate and Git-version-coupled); no native large-object/attachment
  handling (Git's own well-known pain point with large binaries, addressed
  externally via LFS rather than in the core format); no native
  digital-signature-per-object (Git signs commits/tags via GPG as an
  external convention, not a format-level structure).
- **Lessons learned:** content-addressed, hash-sorted index files with
  O(log n) binary-search lookup is precisely what
  [18-container-index.md](18-container-index.md) §2 adopts; append-only,
  layerable pack files directly informed
  [16-native-container-format.md](16-native-container-format.md) §8's
  incremental-update design.
- **Adopted:** content-addressed Block model; sorted-index binary search;
  append-then-later-consolidate lifecycle (Git's "loose objects → packed
  objects → repacked" progression maps directly onto `.oep`'s
  append/repack/defragment operations, [20-package-maintenance.md](20-package-maintenance.md)).
  **Rejected:** delta compression between Blocks (Git's biggest single
  source of format complexity) — v0.1 stores each Block's canonical
  payload directly, optionally Zstd-compressed
  ([17-block-layout.md](17-block-layout.md) §4), accepting a larger file
  size in exchange for dramatically simpler, more independently-
  implementable framing; native, first-class large-object chunking
  instead of Git's external LFS bolt-on ([17](17-block-layout.md) §5).

## LMDB

- **Advantages:** memory-mapped B+tree design gives extremely fast
  random-access reads with true zero-copy semantics; single-writer/
  multi-reader MVCC concurrency model with no separate WAL needed for
  reads; proven low-corruption-risk design (copy-on-write pages, no
  in-place page mutation during a transaction).
- **Limitations:** designed as an embedded key-value engine, not an
  interchange/distribution format — not meant to be emailed, copied
  between organizations, or opened by independent third-party tooling the
  way a `.oep` package must be; page-copy-on-write growth model doesn't
  map cleanly onto "append new content, keep old content immutable
  forever" (LMDB expects old pages to eventually become reclaimable, in
  tension with an engineering record's retention expectations, see
  [12-repository-internals-lifecycle-and-maintenance.md](12-repository-internals-lifecycle-and-maintenance.md)
  §1's deliberately-non-automatic GC).
- **Lessons learned:** mmap-friendly, alignment-conscious physical layout
  meaningfully speeds up local random access with no logical-format cost
  — directly influenced `.oep`'s 64-byte Block alignment
  ([16-native-container-format.md](16-native-container-format.md) §2) and
  the explicit mmap guidance in
  [19-streaming-model.md](19-streaming-model.md) §4.
- **Adopted:** alignment-for-mmap discipline; copy-on-write-style atomic
  update pattern (`.oep`'s "write new, then atomically swap the
  Header/Footer pointer," [16](16-native-container-format.md) §8, mirrors
  LMDB's copy-on-write page commit).
- **Rejected:** LMDB as an interchange format (its embedded-engine
  design goal is fundamentally different from `.oep`'s
  portable-distribution goal — this is why `.oep` is a self-contained
  file format, not a database engine's on-disk representation exposed
  directly).

## Cross-cutting synthesis

No single existing technology solves the combined requirement set (native
streaming *and* random access *and* content addressing *and* built-in
cryptographic integrity *and* portability as a single interchange file).
`.oep` v0.1's design is best understood as: **TAR's sequential
self-description**, **Git packfile's content addressing and sorted
index**, **ZIP's directory-plus-content-region shape (inverted to be
header-forward)**, **SQLite's reference-implementation-as-standard
philosophy**, and **LMDB's alignment/mmap discipline** — combined, with
each technology's specific limitation in this domain deliberately not
carried forward.
