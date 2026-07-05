# ADR-0008: BLAKE3 default with mandatory SHA-256 fallback, per-hash algorithm tagging

Status: Accepted

## Context

Content addressing and integrity verification
(08-integrity-verification-strategy.md) depend on a cryptographic hash
algorithm. A 20+ year format must anticipate algorithm evolution or
deprecation.

## Problem Statement

Which hash algorithm should the Repository use, and how should the format
avoid being permanently locked to whatever choice is made today?

## Alternatives Considered

1. **SHA-256 only.** Maximum ecosystem maturity and universal library
   support, but noticeably slower than modern alternatives at the scale of
   hashing every revision, attachment, and commit in a large repository.
2. **BLAKE3 only.** Excellent speed and strong security margin, but younger
   and less universally implemented than SHA-256; a hard dependency on it
   alone risks the same lock-in problem in the opposite direction if it is
   ever superseded.
3. **BLAKE3 default, SHA-256 mandatory fallback, per-value algorithm
   tagging** so both are always supported and future algorithms can be
   added without invalidating history.

## Selected Solution

Option 3, as specified in 08-integrity-verification-strategy.md §1.

## Rationale

- BLAKE3's speed matters directly at Repository scale — potentially every
  object revision, attachment, commit, and Merkle recomputation during
  Level 1/2 verification (08-integrity-verification-strategy.md §5) is
  hash-bound.
- Requiring every implementation to also support SHA-256 verification
  guarantees a universally-implementable fallback exists even where a
  BLAKE3 library isn't available, satisfying the independent-implementation
  tenet.
- Tagging every stored hash with its algorithm means a future algorithm
  transition never requires rehashing or invalidating existing history —
  old content simply keeps verifying against its original tag forever.

## Consequences

- Every hash value in the format is a tagged pair (`{alg, hex}`), not a
  bare hex string — a small but pervasive format detail every
  implementation must respect from day one.
- Implementations must ship at least two hash algorithm implementations
  (BLAKE3 and SHA-256), a modest but bounded dependency footprint.

## Future Considerations

Adopting a third algorithm later (e.g. in response to a cryptographic
break) only requires: (1) adding it to the supported tag set, (2) updating
the manifest's default recommendation. No existing stored data or past
ADR needs to change.
