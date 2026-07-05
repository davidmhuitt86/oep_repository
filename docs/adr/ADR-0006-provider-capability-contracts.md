# ADR-0006: Four minimal Storage Capability contracts instead of one unified interface

Status: Accepted

## Context

The Repository must never depend on a specific storage technology
(filesystem, SQLite, PostgreSQL, cloud object store, in-memory). Some
interface abstraction is required between Repository core logic and
physical storage.

## Problem Statement

Should storage be abstracted behind one general-purpose interface, or
several small interfaces each matching a distinct access pattern?

## Alternatives Considered

1. **One unified storage interface** covering all needs generically (e.g.
   a generic key-value get/put/list covering blobs, log, refs, and index
   alike).
2. **Four minimal Capability contracts** — `BlobStore` (content-addressed),
   `AppendLog` (ordered append), `RefStore` (mutable pointer with CAS),
   `Index` (best-effort query) — each matching one genuinely distinct
   access pattern (06-storage-provider-abstraction.md §2).

## Selected Solution

Four minimal contracts.

## Rationale

- A single generic interface would either expose least-common-denominator
  semantics (losing the compare-and-swap guarantee RefStore needs for safe
  concurrent branch moves, or the strict ordering AppendLog needs for
  journal replay) or leak storage-specific concepts upward.
- Each contract maps directly onto what a Provider is naturally good at —
  content-addressed blobs, ordered logs, atomic pointers, and best-effort
  query — letting Providers implement only what they're suited for
  (06-storage-provider-abstraction.md §4 provider matrix).
- Small contracts are easier for an independent implementation to build
  and test in isolation, supporting the "independent implementations"
  design tenet.

## Consequences

- Repository core code must compose across four contracts rather than
  calling one interface, adding modest internal complexity in exchange for
  Provider flexibility.
- New Provider implementations only need to satisfy contracts relevant to
  what they back (e.g. a pure object store Provider needs an auxiliary
  ordering mechanism to satisfy `AppendLog`, called out explicitly in
  06-storage-provider-abstraction.md §4 so implementers aren't surprised).

## Future Considerations

If a fifth genuinely distinct access pattern emerges (e.g. streaming large
binary ranges), it should be evaluated the same way: is it truly a new
pattern, or expressible via `BlobStore` with chunking? Only add a new
contract if composition genuinely fails.
