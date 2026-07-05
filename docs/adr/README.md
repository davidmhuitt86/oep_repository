# Architecture Decision Records

This log is part of the permanent engineering record. ADRs are never
edited after acceptance — a changed decision gets a new ADR that
references and supersedes the old one.

| ID | Title | Status |
|---|---|---|
| [0001](ADR-0001-content-addressable-storage.md) | Content-addressable storage for Engineering Objects | Accepted |
| [0002](ADR-0002-object-identity.md) | UUIDv7 permanent identity + hash-chained revisions | Accepted, partially superseded by 0010 |
| [0003](ADR-0003-relationships-as-objects.md) | Relationships are Engineering Objects, not a separate store | Accepted |
| [0004](ADR-0004-commit-groups-events.md) | Commits group Events; they are not one-Event-per-commit | Accepted |
| [0005](ADR-0005-policy-as-capability.md) | Branch protection and validation enforcement are Capabilities, not core logic | Accepted |
| [0006](ADR-0006-provider-capability-contracts.md) | Four minimal Storage Capability contracts instead of one unified interface | Accepted |
| [0007](ADR-0007-package-format.md) | `.oep` is a deterministic content-addressed archive, not a SQLite file | Accepted, re-affirmed by 0013 |
| [0008](ADR-0008-hash-algorithm-agility.md) | BLAKE3 default with mandatory SHA-256 fallback, per-hash algorithm tagging | Accepted |
| [0009](ADR-0009-closed-primitive-set.md) | Five fundamental concepts; no new primitives without exhausting composition first | Accepted, elevated to constitutional status by 0012 |
| [0010](ADR-0010-three-part-identity.md) | Three-part identity: EOID / Revision ID / Content Hash, independent of each other | Accepted (RC2) |
| [0011](ADR-0011-three-layer-separation.md) | Package / Repository / Object Store as three distinct layers | Accepted (RC2) |
| [0012](ADR-0012-five-primitive-constitutional-amendment.md) | Elevate the Five-Primitive Rule to a constitutional principle | Accepted (RC2) |
| [0013](ADR-0013-container-format-reevaluation.md) | Container format re-evaluation — deterministic ZIP retained over a custom OEP container | Accepted (RC2) |
