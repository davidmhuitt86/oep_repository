# Repository Capability Model

Status: Draft v0.1 (Sprint 0.5) · Defines how a Repository advertises what it can do, using the platform's general Capability primitive ([00-repository-architecture.md](00-repository-architecture.md) §3).

## 1. Principle: advertise, never assume

A client of a Repository (an Application, a sync peer, a tool opening a
`.oep` package) must not assume a Repository supports branching,
signing, multi-user sync, or any other behavior. It queries the
Repository's advertised **Repository Capabilities** first. This is the
same Capability concept already used for Storage Providers
([06-storage-provider-abstraction.md](06-storage-provider-abstraction.md))
and for policy enforcement ([ADR-0005](../adr/ADR-0005-policy-as-capability.md)),
applied one level up: to the Repository *instance* itself, not just to
what backs it.

## 2. Capability record

```
RepositoryCapabilities {
  format_version        : semver    // 01-repository-specification.md §3
  container_format       : { major: uint16, minor: uint16 }  // 16-native-container-format.md §4
  capabilities           : [CapabilityFlag]
}

CapabilityFlag {
  id      : string     // reverse-DNS namespaced, e.g. "oep.repo.Branching"
  version : uint16
  status  : enum        // Supported | Unsupported | ReadOnly | Partial
}
```

## 3. Standard capability identifiers (v0.1)

| Capability ID | Meaning |
|---|---|
| `oep.repo.Branching` | Supports creating/switching branches ([04-branch-architecture.md](04-branch-architecture.md)) |
| `oep.repo.Merging` | Supports structural merge commits ([04-branch-architecture.md](04-branch-architecture.md) §3) |
| `oep.repo.OfflineOperation` | Fully functional with no network access (true for any conformant local Storage Provider by construction, but advertised explicitly so a client never has to guess) |
| `oep.repo.Synchronization` | Supports the sync model in [26-synchronization-philosophy.md](26-synchronization-philosophy.md) |
| `oep.repo.CryptographicSigning` | Can produce and verify signed Commits/Packages ([08-integrity-verification-strategy.md](08-integrity-verification-strategy.md) §3) |
| `oep.repo.LargeObjectSupport` | Supports chunked Attachments beyond a given size ([17-block-layout.md](17-block-layout.md) §5) |
| `oep.repo.ImmutableArchiveMode` | Repository is permanently sealed — see [23-repository-lifecycle.md](23-repository-lifecycle.md) `Archived` state |
| `oep.repo.ReadOnlyMode` | No mutation accepted regardless of caller identity — distinct from `ImmutableArchiveMode`: read-only can be lifted, archive mode is a terminal-leaning state |
| `oep.repo.MultiUserCollaboration` | Supports concurrent-editing semantics ([27-multi-user-behavior.md](27-multi-user-behavior.md)) |
| `oep.repo.DistributedSynchronization` | Supports peer-to-peer, non-hub-and-spoke synchronization topologies ([26-synchronization-philosophy.md](26-synchronization-philosophy.md) §4) |

This list is open, not closed — new capability IDs may be introduced by
any namespace without a format version bump, exactly like Engineering
Object types ([02-object-storage-architecture.md](02-object-storage-architecture.md)
§6). A client encountering an unrecognized capability ID simply cannot
rely on it; it does not fail to open the Repository.

## 4. Why capabilities, not version numbers, gate behavior

**Alternative considered:** gate behavior purely on `format_version`
(e.g. "branching requires format 1.2+"). Rejected as the sole mechanism —
it forces every optional behavior into the same linear version sequence,
so a Repository that deliberately disables signing (e.g. an
`ImmutableArchiveMode` export with signatures stripped for a public
release) would have no way to express that except by *lying* about its
format version. Capabilities decouple "what format bytes can this reader
parse" (format_version, a parsing concern) from "what behavior does this
specific Repository instance support or allow right now" (capabilities, a
behavioral/policy concern) — consistent with keeping policy enforcement
out of core format mechanics ([ADR-0005](../adr/ADR-0005-policy-as-capability.md)).

## 5. Discovery

Capability advertisement is read from the Manifest Block
([17-block-layout.md](17-block-layout.md) §6) for a packaged `.oep`, or
queried live via the Repository API for an open, in-use Repository. A
client MUST check capabilities before attempting an operation that
depends on one (e.g. checking `oep.repo.Merging` before offering a merge
UI) rather than attempting the operation and handling failure — this
keeps capability-gated behavior discoverable and self-documenting rather
than trial-and-error.
