# OEPFS — Open Engineering Platform File System (Exploratory)

Status: **Exploratory discussion document only.** No implementation
decisions are made here. Nothing in this document is normative, and
nothing here is a commitment to build OEPFS. It exists to think through
the concept before any Sprint schedules it.

## 1. The concept

Today, the Repository stores Engineering Objects, not files
([00-repository-architecture.md](00-repository-architecture.md) §2). But
every existing tool a user already knows — Explorer, Finder, a terminal,
a CAD tool's "open file" dialog — expects a filesystem-shaped view of
data: folders, files, paths. OEPFS explores whether the Repository could
expose the engineering graph *as if* it were a filesystem, where:

- Traditional "folders" become **semantic views** — the result of a
  standing graph query (e.g. "all Components that are `part_of` this
  Assembly"), not a physical grouping decision made once at file-creation
  time.
- "Files" a user sees when browsing are Engineering Objects (or specific
  revisions of them) presented through a materialization layer.
- Organization happens through **relationships, tags, and views**
  instead of a single fixed folder hierarchy — the same Engineering
  Object can appear in many "folders" simultaneously (e.g. under
  `by-assembly/`, `by-type/Sensor/`, `by-owner/acme/`) because they're all
  just different queries over the same graph, not different physical
  locations.

## 2. Relationship to the Engineering Repository

OEPFS would be a **presentation layer strictly above** the Repository —
consistent with the layering in
[10-layered-architecture.md](10-layered-architecture.md), it would sit
alongside Engineering Studio and other Applications, consuming the
Repository API and never bypassing it. OEPFS is not a fourth architectural
layer of the Repository itself; it is a client, likely implemented as a
FUSE-style (or platform-equivalent) virtual filesystem driver that
translates filesystem syscalls (`readdir`, `open`, `read`) into Repository
queries (Index queries over Relationships, `BlobStore` reads over content
hashes). This is why the exploration lives in the architecture set but is
explicitly non-normative for the Repository itself — the Repository needs
to change nothing to make OEPFS possible; OEPFS is purely a consumer.

## 3. Advantages

- **Zero-retraining onboarding** — engineers already know how to use a
  filesystem; OEPFS would let existing tools (that only know how to
  "open a file") interoperate with the graph without every single tool in
  an organization's toolchain needing a native Repository integration.
- **Multiple simultaneous organizations of the same data** — the same
  Component can appear under `by-assembly/`, `by-supplier/`, `by-status/`
  views without duplication, because each view is a query, not a copy.
- **Views can evolve without data migration** — reorganizing "folders"
  is reorganizing queries/tags, not moving files, so it never risks the
  data itself and is trivially reversible.
- **Read-side gateway for legacy/third-party tools** — a CAD tool that can
  only "open a file from disk" can, through OEPFS, transparently read
  Engineering Object content without OEP needing a plugin for every tool
  in existence.

## 4. Disadvantages

- **Write semantics are the hard problem.** Filesystems assume "save
  overwrites this file"; the Repository assumes "every change is a new
  revision inside a Commit inside a Transaction"
  ([03](03-commit-architecture.md), [05](05-transaction-journal-architecture.md)).
  A naive OEPFS write path (a tool does `open(O_WRONLY)`, writes, closes)
  has no natural mapping to "which Commit does this belong to, and when is
  it committed" — a single filesystem `write()` cannot express engineering
  intent (a commit message, a batch of related changes) the way an
  explicit `oep commit` can. Getting this wrong risks silently coercing
  the Repository's rich change model down to "one file write = one
  auto-commit," discarding exactly the atomicity and intentionality the
  Commit model was designed to guarantee ([03](03-commit-architecture.md) §1).
- **Performance mismatch** — `readdir` over a semantic view backed by a
  graph query is not the O(1) directory-entry-list a filesystem API
  implies; large or expensive views could make OEPFS feel slow or
  inconsistent compared to a real filesystem, especially under tools that
  poll directories aggressively.
- **Ambiguous identity under filesystem semantics** — a file has one path;
  an Engineering Object can appear at many OEPFS paths simultaneously.
  Tools that assume "a file's path is its identity" (e.g. some CAD tools'
  reference-resolution logic) could break in surprising ways when the
  "same" object is opened from two different semantic paths.
- **Caching and staleness** — a real filesystem's contents don't change
  underneath an open file handle in ways users don't expect; a
  graph-query-backed view could change (a Component leaves an Assembly)
  while a tool has it "open," with no standard filesystem notification
  users' tools are built to expect.

## 5. Alternative approaches

1. **Read-only OEPFS** (mounted view for browsing/opening only; all writes
   go through the native Repository API/CLI, never through the mounted
   filesystem). Sidesteps the hardest problem (§4, write semantics)
   entirely, at the cost of not being a full filesystem replacement.
2. **Staged-write OEPFS** — writes through the mount land in a `workspace`
   staging area ([01-repository-specification.md](01-repository-specification.md)
   §2) exactly like CLI-staged changes, requiring an explicit
   commit action (via a companion CLI/UI, not a filesystem call) before
   becoming permanent. Preserves the Commit model's intentionality while
   still allowing arbitrary tools to write bytes into the staging area.
3. **No filesystem projection; native app integrations only** — skip
   OEPFS, invest instead in plugins/exporters for the small number of
   tools that matter most. Lower architectural risk, higher integration
   cost per tool, and no story for arbitrary/future third-party tools.
4. **Export-based synchronization** (a materialized, periodically
   refreshed folder tree, explicitly a snapshot/cache, not a live mount).
   Avoids live write-semantics problems entirely at the cost of staleness
   and no true bidirectional flow.

The **read-only or staged-write** variants (1 or 2) look like the most
promising starting points if this is ever pursued, precisely because they
avoid silently degrading the Commit model's guarantees.

## 6. Enterprise implications

- **Access control** — a semantic view can expose Engineering Objects a
  user shouldn't see if view generation doesn't respect the same
  authorization model the native Repository API enforces; OEPFS must not
  become a side channel that bypasses whatever access-control Capability
  governs the underlying Repository.
- **Audit** — every OEPFS-mediated read/write must still produce the same
  auditable Events/Commits as a native API call; a filesystem mount must
  not become an unaudited path into the engineering record.
- **Tool certification** — organizations may need to certify which
  third-party tools are approved to write through OEPFS, since a poorly
  behaved tool could stage broken or partial data.

## 7. Synchronization implications

A live OEPFS mount reflecting a Repository that is simultaneously being
synchronized with remote peers (a capability noted as future work in
[05-transaction-journal-architecture.md](05-transaction-journal-architecture.md)
§4) raises a real question: does the mounted view update live as sync
happens? A conservative starting design would present a **consistent
snapshot per mount session** (view reflects the Repository state as of
mount time or explicit refresh), avoiding the substantially harder problem
of a filesystem view that changes out from under an open tool mid-session.

## 8. Offline behavior

OEPFS mounted against a Repository with a purely local Storage Provider
([06-storage-provider-abstraction.md](06-storage-provider-abstraction.md))
works identically offline or online, since it never depends on network
access beyond whatever the underlying Repository already requires — this
is a direct benefit of OEPFS being a pure client of the Repository API
rather than a parallel system with its own connectivity requirements.

## 9. Where this stands

This is Sprint 0 exploration only. No Milestone references OEPFS. Before
any implementation work is scheduled, the write-semantics question (§4,
§5) needs a dedicated ADR-level decision — likely "read-only or
staged-write only," given how directly a naive live-write mount would
conflict with the Commit model this entire specification set is built
around.
