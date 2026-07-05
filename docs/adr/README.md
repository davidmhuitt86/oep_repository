# Architecture Decision Records

This log is part of the permanent engineering record. ADRs are never
edited after acceptance — a changed decision gets a new ADR that
references and supersedes the old one.

| ID | Title | Status |
|---|---|---|
| [0001](ADR-0001-content-addressable-storage.md) | Content-addressable storage for Engineering Objects | Accepted |
| [0002](ADR-0002-object-identity.md) | UUIDv7 permanent identity + hash-chained revisions | Accepted |
| [0003](ADR-0003-relationships-as-objects.md) | Relationships are Engineering Objects, not a separate store | Accepted |
| [0004](ADR-0004-commit-groups-events.md) | Commits group Events; they are not one-Event-per-commit | Accepted |
| [0005](ADR-0005-policy-as-capability.md) | Branch protection and validation enforcement are Capabilities, not core logic | Accepted |
| [0006](ADR-0006-provider-capability-contracts.md) | Four minimal Storage Capability contracts instead of one unified interface | Accepted |
| [0007](ADR-0007-package-format.md) | `.oep` is a deterministic content-addressed archive, not a SQLite file | Accepted |
| [0008](ADR-0008-hash-algorithm-agility.md) | BLAKE3 default with mandatory SHA-256 fallback, per-hash algorithm tagging | Accepted |
| [0009](ADR-0009-closed-primitive-set.md) | Five fundamental concepts; no new primitives without exhausting composition first | Accepted |
