# Package Maintenance — `.oep` v0.1

Status: Draft v0.1 (Sprint 0.5) · Maintenance operations over the native container, complementing the logical-layer maintenance in [12-repository-internals-lifecycle-and-maintenance.md](12-repository-internals-lifecycle-and-maintenance.md).

## 1. Optimization

Container-level optimization targets physical layout, never logical
content:

- **Reordering Blocks** so frequently-accessed-together Blocks (e.g. a
  Commit and the Object Blocks it references) sit near each other,
  reducing seek distance for common access patterns.
- **Recompressing** Blocks with a newer/better codec as
  [17-block-layout.md](17-block-layout.md) §4 codec support evolves,
  always re-verifying `content_hash` against the recompressed Block's
  decompressed bytes before replacing it.
- **Rebuilding the Container Index** in sorted order after
  incremental appends ([16-native-container-format.md](16-native-container-format.md)
  §8) have produced multiple superseded Index fragments.

Optimization never changes any `content_hash` or logical identity — it is
purely a rewrite of physical layout, safe to run repeatedly and safe to
skip entirely (a never-optimized package is fully valid, just less
locality-efficient).

## 2. Repacking

Repacking rewrites a `.oep` file from scratch: read every live Block via
the current Index, write a fresh Header/Block Stream/Index/Footer with no
dead space from prior incremental updates or superseded Index fragments
([16-native-container-format.md](16-native-container-format.md) §8).
Repacking is the container-format equivalent of Git's `git gc` packing
step. It:

1. Reads the current Index.
2. Writes a new file (never in place — atomic rename on completion,
   avoiding any window where a crash could leave a half-repacked file
   claiming to be the canonical package).
3. Verifies the new file's full integrity chain
   ([18-container-index.md](18-container-index.md) §4) before the rename.
4. Only then replaces the original.

## 3. Defragmentation

Distinct from repacking: defragmentation reclaims dead space (padding
from removed/superseded Blocks, stale Index fragments left by incremental
updates) **without** necessarily reordering live Blocks for locality —
a cheaper, more frequent operation than full repacking, appropriate as a
lighter-weight periodic maintenance step. Defragmentation is repacking's
strict subset: every defragmentation is safe to implement as "repack, but
skip the reordering-for-locality pass."

## 4. Validation

Container-level validation is the physical-format instance of
[08-integrity-verification-strategy.md](08-integrity-verification-strategy.md)
§5's Level 0/1, expressed in container terms:

| Level | Container check |
|---|---|
| 0 | Header/Footer magic and version checks, `header_hash` and `index_hash` recompute correctly ([18-container-index.md](18-container-index.md) §4) |
| 1 | Every `IndexEntry`'s referenced Block recomputes to its declared `content_hash` |

Failing Level 0 triggers Index rebuild (§ [18](18-container-index.md) §5)
before anything else is attempted — a corrupt Index must not be trusted
even to decide what Level 1 needs to check.

## 5. Upgrade

A container format minor-version upgrade (new optional Block types, new
Header flags) requires no rewrite — old Blocks remain valid as-is; new
optional structures are additive (§ [16](16-native-container-format.md)
§7). A major-version upgrade requires a full repack under the new
Header/Block framing rules, treated exactly like the logical-layer format
upgrade in
[12-repository-internals-lifecycle-and-maintenance.md](12-repository-internals-lifecycle-and-maintenance.md)
§4 — an explicit, auditable, resumable Operation, never an implicit
side effect of opening a file.

## 6. Repair

| Detected condition | Repair strategy |
|---|---|
| Index missing/corrupt, all Blocks intact | Rebuild Index by scanning the Block Stream ([18-container-index.md](18-container-index.md) §5) |
| Single Block's `content_hash` mismatch, good copy available elsewhere (peer, prior backup) | Replace only that Block (via incremental update, §16 §8), re-verify, never silently patch bytes without re-verification |
| Footer missing/truncated (file cut short) | If the Header's own `index_offset`/`index_length` are intact, the Footer is redundant metadata (§ [16](16-native-container-format.md) §6) and can be regenerated; if the Header itself is damaged, the file is unrecoverable from the container layer alone and must fall back to a peer/backup copy |
| Unrecoverable content, no good copy available anywhere | Recorded as a permanent tombstone at the logical layer ([12-repository-internals-lifecycle-and-maintenance.md](12-repository-internals-lifecycle-and-maintenance.md) §3) — the container-format layer's role ends at "this Block cannot be recovered," it does not itself decide how the engineering record should represent that gap |

## 7. Recovery

Recovery is the end-to-end procedure combining §4–§6: validate, identify
the first failure, apply the narrowest applicable repair, re-validate, and
only escalate to a full repack (§2) if narrow repair is insufficient.
Recovery is always logged as an auditable Operation
([05-transaction-journal-architecture.md](05-transaction-journal-architecture.md)),
consistent with the logical layer's repair philosophy of never silently
rewriting history to erase evidence that something went wrong
([12-repository-internals-lifecycle-and-maintenance.md](12-repository-internals-lifecycle-and-maintenance.md)
§3).
