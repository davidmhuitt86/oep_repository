# Package Format Proposal — `.oep`

Status: Draft v0.1 · Proposal, open for revision before Milestone 1 freeze.

## 1. Goal

`project.oep` is what a user sees as "one project." Internally it must
contain a complete, self-sufficient snapshot of a Repository's structured
engineering data — objects, relationships, events, commits, branches,
integrity metadata — sufficient to `open` it as a fully functional
Repository on any conformant implementation, on any platform, with no
external dependency.

## 2. Candidate formats and trade-off analysis

| Option | Portability | Diffability | Streamability | Tool dependency | Long-term risk |
|---|---|---|---|---|---|
| Single SQLite file | Excellent (one file, ubiquitous library) | Poor — binary format, byte-level diffs are meaningless even for a one-object change | Poor — must open whole file, no streaming import/export | Requires a SQLite implementation in every conformant tool | SQLite's on-disk format is stable and well-documented, but coupling the *package format* to it means every future implementation's packaging code must embed or link SQLite, an implementation dependency the format itself shouldn't force |
| Raw directory tree (uncompressed, mirrors the logical layout) | Poor as a single artifact — "one file" requirement fails outright | Excellent — plain content-addressed files, trivially diffable/mergeable with generic tools | Excellent | None beyond a filesystem | Best transparency, worst as a distributable package |
| **Deterministic content-addressed archive (ZIP-based, uncompressed store method for objects, canonical entry ordering)** | Excellent — one file, ZIP is universally supported | Good — each entry is independently addressable; entry-level diff tools work even though the container is binary | Good — ZIP supports streaming random access to individual entries without decompressing the whole archive | Only requires a ZIP reader/writer, present in essentially every language's standard library | Selected |
| Custom binary container (bespoke framing) | Excellent | Depends entirely on the custom spec | Can be designed for it | Requires every implementation to build a parser from scratch — highest risk for a 20-year format with no existing tooling ecosystem | Rejected — reinvents what ZIP already solves, with no compensating benefit |

**Recommendation: the deterministic content-addressed archive**, specified
as **OEP Package Format v1**, selected over both SQLite-as-package and a
custom binary format. See
[ADR-0007](../adr/ADR-0007-package-format.md) for full rationale.

## 3. Structure

A `.oep` file is a ZIP archive (uncompressed/`STORE` method for content
entries so that content hashes can be verified by reading raw bytes
without a decompression step; `DEFLATE` permitted only for the manifest and
index entries, which are not content-addressed) with these entries:

```
project.oep  (ZIP container)
├── manifest.json                  # Repository manifest, see 01-repository-specification.md §3
├── objects/<hash-prefix>/<hash>    # content-addressed object revisions
├── relationships/<hash-prefix>/<hash>
├── events/segment-<n>.log
├── commits/<commit_id>
├── branches/<name>.ref
├── attachments/<hash-prefix>/<hash>
├── integrity/merkle-roots.json     # see 08-integrity-verification-strategy.md
└── integrity/signatures/<commit_id>.sig
```

Entries are written in a **canonical order** (lexicographic by full entry
path) with a **fixed timestamp** (e.g. epoch zero) on every entry, so that
packaging the same Repository state twice, on different machines, produces
a byte-identical `.oep` file. This determinism is required for the
Repository's conformance guarantees ([01](01-repository-specification.md)
§5) to extend to the packaged form, and is what makes a package's own hash
a meaningful integrity check.

## 4. Pack / unpack semantics

- **Pack** (`oep pack`, see [09](09-cli-specification.md)) walks the
  logical stores via the bound Storage Providers' Capability contracts
  ([06](06-storage-provider-abstraction.md)) and writes the archive. Pack
  never depends on which Provider is bound — it reads through the same
  contracts application code uses.
- **Unpack** (`oep unpack`, or `open` directly against a `.oep` file) can
  either materialize the archive into a working Repository backed by any
  chosen Provider, or — for read-only inspection — a Provider MAY treat
  the archive itself as a valid (read-only) backing store by implementing
  `BlobStore`/`AppendLog`/`RefStore` directly against ZIP entry reads. This
  second option is what makes ".oep is a portable repository," not just
  "an export format" — an `.oep` file can be `open`ed in place.

## 5. Partial and incremental packages (deferred)

A full package always contains complete history. Later phases may define
a **shallow package** (only the state at one commit, no ancestor history)
or a **delta package** (journal segments since a known checkpoint, see
[05-transaction-journal-architecture.md](05-transaction-journal-architecture.md)
§4) for bandwidth-constrained sync. Both are out of scope for Milestone 1
and MUST be distinguishable from a full package via a manifest field
(`package_kind: full | shallow | delta`) so that no tool ever silently
treats a partial package as a complete engineering record.

## 6. Signing the package as a whole

In addition to per-commit signatures ([08](08-integrity-verification-strategy.md)),
the package's own top-level `integrity/merkle-roots.json` MAY carry a
detached signature over the whole archive's canonical content, letting a
recipient verify "this exact package, as a unit, was produced/endorsed by
X" independent of verifying each commit individually.
