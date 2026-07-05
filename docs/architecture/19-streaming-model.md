# Streaming Model — `.oep` v0.1

Status: Draft v0.1 (Sprint 0.5) · Specifies how the native container is consumed without requiring full in-memory materialization.

## 1. Sequential reading

A reader with no random access (e.g. consuming from a pipe or an
append-only network stream) can process a `.oep` file in one forward
pass:

1. Read and validate the 256-byte Header ([16-native-container-format.md](16-native-container-format.md) §4).
2. Read Blocks sequentially from offset 512; each `BlockHeader` declares
   `payload_length`, so the reader always knows exactly how far to
   advance to reach the next Block, with no lookahead or backtracking
   required.
3. Optionally verify each Block's `content_hash` as it arrives (streaming
   hash computation, matching
   [11-repository-internals-storage-and-encoding.md](11-repository-internals-storage-and-encoding.md)
   §5's incremental-hashing approach).
4. Upon reaching the Container Index and Footer, perform final
   whole-package verification (§ [18-container-index.md](18-container-index.md) §4).

This is the mode a naive `cat package.oep | oep import` pipeline would
use, and it never requires seeking — the direct fix for the weakness ZIP
had under ADR-0013's own analysis.

## 2. Partial downloads / progressive loading

Because the Header declares `index_offset`/`index_length` immediately
(§ [16](16-native-container-format.md) §4), a reader with a partial
download (e.g. the first N megabytes of a large package fetched over an
unreliable or resumable connection) can:

- Validate the Header alone as soon as the first 256 bytes arrive.
- Begin processing whichever Blocks have already been received, in
  order, without waiting for the rest of the file — a partial view of a
  large repository is immediately useful (e.g. showing the manifest and
  the first N commits while the rest streams in).
- If the Index has not yet arrived, defer only Index-dependent operations
  (arbitrary random-access lookup by Content Hash) — everything
  achievable via sequential Block processing (§1) remains available.

This directly serves "partial loading" and "future distributed
repositories" without requiring a special "partial package" format
variant — partiality is a property of *how much of the same file* has
arrived, not a different file structure.

## 3. Remote repositories

For a `.oep` file served over HTTP(S) or an equivalent range-request-capable
transport, the Header-forward layout enables a **range-request access
pattern** functionally equivalent to remote ZIP central-directory access,
but front-loaded:

1. `GET` bytes `[0, 256)` — Header, including `index_offset`/`index_length`.
2. `GET` bytes `[index_offset, index_offset + index_length)` — the
   Container Index, without downloading any Block content.
3. For any Content Hash of interest, consult the now-local Index for its
   `block_offset`/`block_length`, then `GET` exactly that byte range.

This gives remote, lazy, Block-granular access with exactly 2 round trips
before the first targeted Block fetch (vs. ZIP's typical "fetch the last N
bytes for the central directory, then targeted fetches" — comparable
shape, but the native format's Index is not tied to file-end conventions
and can be relocated or duplicated for redundancy in future minor
versions without changing this access pattern).

## 4. Memory mapping

A local reader MAY memory-map the entire file (or just the Block Stream
region) and treat `IndexEntry.block_offset` values as direct offsets into
the mapped region, avoiding an explicit read syscall per Block —
consistent with the mmap guidance in
[11-repository-internals-storage-and-encoding.md](11-repository-internals-storage-and-encoding.md)
§6. The 64-byte Block alignment (§ [16](16-native-container-format.md) §2)
is chosen partly to keep mmap'd Block boundaries cache-line-friendly.
mmap is a local-only optimization; it is never assumed available for
remote packages (§3), which fall back to explicit range requests.

## 5. Lazy loading

Opening a `.oep` file for use as a live Repository ([09-cli-specification.md](09-cli-specification.md)
`oep unpack`/in-place-open) never requires materializing every Block's
payload up front:

- The Container Index is read fully (it's small relative to package size
  — one `IndexEntry` per Block, not per byte of content).
- Object/Relationship/Commit/Branch Block payloads are deserialized lazily,
  on first logical access, exactly mirroring
  [11-repository-internals-storage-and-encoding.md](11-repository-internals-storage-and-encoding.md)
  §8's lazy-loading principle for the logical Object Store.
- Attachment Blocks (potentially chunked, [17-block-layout.md](17-block-layout.md)
  §5) are streamed on demand, chunk by chunk, never fully buffered unless
  the consumer explicitly requests the complete payload.

## 6. Progressive integrity feedback

A streaming consumer does not need to choose between "verify nothing
until the whole file arrives" and "block until fully verified." Verification
is itself progressive: each Block's `content_hash` is checkable the moment
that Block finishes arriving (§1), giving early corruption detection long
before the Index/Footer are reached — important for large repositories
where waiting for a full-file hash before surfacing any error would be a
poor experience matching none of the "large engineering repositories" or
"efficient synchronization" design goals.
