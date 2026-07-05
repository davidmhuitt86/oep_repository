# Layered Architecture — Package, Repository, Object Store

Status: Draft v0.1 (RC2) · Makes explicit a three-layer separation that RC1 left implicit inside a single "Repository" concept.

## 1. Why this separation is needed

RC1 described "the Repository" as one box containing objects,
relationships, events, commits, branches, journal, and packaging together
(00-repository-architecture.md §4). Sprint 0 review found this conflates
three genuinely different concerns with different lifecycles, different
consumers, and different rates of change:

- Packaging changes when distribution/exchange needs change.
- Version history (commits/branches/sync) changes when collaboration
  workflow needs change.
- Object/relationship/event storage changes when the engineering data
  model itself changes.

Collapsing these into one undifferentiated "Repository" makes it easy to
accidentally couple packaging concerns to storage concerns, or history
concerns to object-model concerns — exactly the kind of hidden coupling
the constitutional principles warn against. RC2 makes the three layers
explicit.

## 2. The three layers

```
┌──────────────────────────────────────────────────────────┐
│  Engineering Package (.oep)                                │
│  Portable exchange format · distribution · signing ·        │
│  import/export · archival                                   │
│  → 07-package-format-proposal.md                             │
├──────────────────────────────────────────────────────────┤
│  Engineering Repository                                      │
│  Version history · commits · branches · synchronization ·   │
│  transaction journal · repository management                │
│  → 03-commit-architecture.md, 04-branch-architecture.md,     │
│    05-transaction-journal-architecture.md                    │
├──────────────────────────────────────────────────────────┤
│  Engineering Object Store                                    │
│  Persistent Engineering Objects · relationships · events ·   │
│  internal indexing · revision storage                        │
│  → 02-object-storage-architecture.md                          │
└──────────────────────────────────────────────────────────┘
                        │
                        │ Storage Capability contracts
                        ▼
              Storage Providers (06-storage-provider-abstraction.md)
```

Each layer depends only on the layer(s) below it, never sideways or
upward:

| Layer | Depends on | Never depends on |
|---|---|---|
| Engineering Package | Engineering Repository (it packages a repository's state) | Storage Providers directly — packaging reads through the Repository layer's API, not around it |
| Engineering Repository | Engineering Object Store (commits reference object/relationship revisions; the journal orders mutations to it) | Package format details — a Repository has no notion of "am I inside a .oep right now" |
| Engineering Object Store | Storage Capability contracts (06) | Commit/branch/journal concepts — the Object Store has no notion of "what commit am I in," only of revisions and the commit_id a revision references as external provenance |

## 3. What changes in practice from RC1

- **00-repository-architecture.md**'s single "Repository" box is now
  understood as the middle layer only; Object Store and Package are
  siblings above and below it, not internal subsections of it.
- **01-repository-specification.md**'s logical layout (§2) maps
  `objects/`, `relationships/`, `events/` to the Object Store layer;
  `commits/`, `branches/`, `journal/` to the Repository layer; and
  everything under [07-package-format-proposal.md](07-package-format-proposal.md)
  to the Package layer. `index/` and `workspace/` remain derived caches
  that may span layers (an index can serve Object Store queries and
  Repository-level history queries alike) without becoming a fourth
  layer themselves.
- **A Repository can, in principle, back multiple Object Stores or vice
  versa** is explicitly *not* a Milestone 1 concern — Milestone 1 assumes
  a 1:1 pairing (one Repository, one Object Store, packaged as one
  `.oep`). The layering is introduced now specifically so that a future
  need to decouple them (e.g. one Object Store shared read-only across
  multiple Repository histories, relevant to Engineering Exchange) does
  not require a retroactive architecture change — only a relaxation of a
  cardinality assumption already isolated to its own layer.

## 4. Non-goal

This layering is an architectural clarification, not a mandate that a
conformant implementation ship as three separate deployable components.
A single implementation binary may implement all three layers internally;
what conformance requires is that the *interfaces between them* are
respected (no packaging logic reaching around the Repository layer to
touch Object Store internals directly, no Repository logic assuming a
specific package container format). See
[ADR-0011](../adr/ADR-0011-three-layer-separation.md).
