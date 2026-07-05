# Repository Architecture

Status: Draft v0.1 · Phase 1 · Normative for terminology, informative for rationale

## 1. Purpose

The Repository is the foundation layer of the Open Engineering Platform. It
is the only component every other component depends on. Nothing above it
(Engineering Studio, Canon, Engineering Exchange, the Device Framework, the
Simulation Engine, the AI Engine, the Document Engine) may bypass it to
touch storage directly. This document defines what the Repository is, what
it is not, and the five concepts everything else is built from.

## 2. What the Repository is not

| Not this | Why the distinction matters |
|---|---|
| A database | Databases model rows and tables. The Repository models engineering knowledge — objects, relationships, history — as first-class citizens, not as an application layered on generic tables. |
| Git | Git versions files and byte blobs with no concept of engineering semantics (typed objects, typed relationships, validation results). The Repository borrows Git's DAG/content-addressing *ideas*, not its file-oriented model. |
| A filesystem | Filesystems have no identity model, no relationship model, no transactional multi-object commit, and no built-in integrity chain. |

The Repository is a fourth thing: an **Engineering Repository** — a
transactional, versioned, content-addressed store of a typed knowledge
graph, with storage mechanics fully abstracted behind providers.

## 3. The five fundamental concepts

Per the platform's constitutional philosophy, the Repository is built from
the smallest possible set of primitives. Every future feature must be
expressible in terms of these five, not as a bespoke sixth concept.

1. **Engineering Object (EO)** — anything that exists in the engineering
   graph: a component, a document, a measurement, an AI analysis, a rule.
   Has permanent identity, mutable content via revisions.
2. **Relationship** — a typed, directed edge between two Engineering
   Objects. A Relationship *is itself* an Engineering Object (it has
   identity, revisions, and can be the subject of further relationships),
   so no separate storage subsystem is needed for it beyond the Object
   Store plus a relationship index.
3. **Operation** — an executable unit of engineering work (create object,
   run simulation, measure voltage). Operations are requests; they are not
   stored directly, they *produce* Events.
4. **Event** — the immutable record that an Operation occurred and what it
   changed. Events are the atomic unit of history. Commits group Events;
   they do not replace them.
5. **Capability** — a named, versioned contract that a Provider
   implements and an application consumes. Storage engines, signature
   schemes, and even future compute backends (simulation, AI) are
   Capabilities from the perspective of anything above the Repository.
   Within the Repository itself, the **Storage Capability family** is the
   one that matters (see [06](06-storage-provider-abstraction.md)).

Everything else defined in this specification set — Commits, Branches, the
Transaction Journal, Indexes, Packages — is a *composition* of these five,
never a sixth foundational primitive. When a future need appears to require
a new fundamental concept, the correct response is to ask whether it can be
expressed as an Engineering Object with a new type, a new Relationship
type, or a new Capability — and only add a true new primitive if all three
fail (see [ADR-0009](../adr/ADR-0009-closed-primitive-set.md)).

## 4. System boundary

The Repository's contract is the only thing every future component must
agree on. Everything on the right is a *client* of the Repository, never a
peer with private access to its storage.

```
                         ┌───────────────────────────────────────┐
                         │      Applications / Higher Layers      │
                         │  Engineering Studio · Canon · Exchange │
                         │  Device Framework · Simulation · AI ·  │
                         │  Document Engine                       │
                         └───────────────────┬─────────────────────┘
                                             │ Repository API
                                             │ (Objects, Relationships,
                                             │  Operations, Events,
                                             │  Commits, Branches)
                         ┌───────────────────▼─────────────────────┐
                         │              OEP Repository              │
                         │  Object Store · Relationship Index       │
                         │  Event Store · Commit Store · Branches   │
                         │  Transaction Journal · Search Indexes    │
                         │  Integrity / Signing · Packaging         │
                         └───────────────────┬─────────────────────┘
                                             │ Storage Capability
                                             │ contracts (BlobStore,
                                             │ AppendLog, RefStore, ...)
                         ┌───────────────────▼─────────────────────┐
                         │           Storage Providers               │
                         │  Filesystem · SQLite · PostgreSQL ·       │
                         │  Cloud Object Store · In-Memory · future  │
                         └───────────────────────────────────────────┘
```

The Repository explicitly does **not** do rendering, simulation, AI
inference, GUI, diagram editing, or device protocol handling (Bluetooth,
CAN). Those are Operations implemented by higher layers that *read and
write through* the Repository API; the Repository only guarantees that
whatever they write is identified, versioned, related, committed, and
verifiable.

## 5. Design tenets (binding on every document in this set)

- **Specification before implementation.** Every mechanism described here
  must be precise enough that an independent team, with no access to any
  reference implementation, could build a byte-compatible Repository.
- **No storage lock-in.** Nothing in the Object model, Commit model, or
  Branch model may name a specific storage technology. If a document in
  this set mentions "the filesystem provider," that is one illustrative
  provider, never the specification.
- **Determinism.** Given the same sequence of Operations, two independent
  implementations must produce byte-identical content hashes and commit
  IDs. This is what makes cross-implementation verification and
  synchronization possible.
- **Immutability of history.** Nothing already committed is ever rewritten
  in place. Corrections are new Events in new Commits. (Branch pointers
  move; the objects they point to are never mutated.)
- **Small closed primitive set.** See §3. Extensibility comes from new
  Object types, Relationship types, and Capabilities — not from new core
  mechanisms.

## 6. Relationship to Milestone 1

Milestone 1 (see [09-cli-specification.md](09-cli-specification.md)) only
exercises a subset of this architecture: object creation, relationship
creation, commit, log, and state reconstruction, all through a CLI, all
against a single local provider. Every mechanism Milestone 1 uses must
already conform to the full specification — there is no "simplified mode"
of the format. What's deferred to later milestones is *scope* (no
merge/branch policy engine, no remote sync, no query language), never
*compatibility*.
