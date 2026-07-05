# Integrity Verification Strategy

Status: Draft v0.1 · Defines hashing, the Merkle structure, signatures, and verification levels.

## 1. Hash algorithm

**Primary: BLAKE3.** Trade-off analysis:

| Option | Speed | Security margin | Ecosystem maturity (as of 2026) | Verdict |
|---|---|---|---|---|
| SHA-256 | Moderate | Excellent, decades of scrutiny | Universal, in every standard library | Retained as **mandatory fallback algorithm** — every implementation MUST support verifying SHA-256-hashed content even if it defaults to BLAKE3 for new writes |
| BLAKE3 | Very high (parallelizable, SIMD-friendly) | Strong, built on well-analyzed primitives (BLAKE2/2s + Bao tree mode) | Growing rapidly, multiple independent implementations exist | **Selected as default** |
| SHA-3 | Moderate | Excellent | Universal but slower than both above with no compensating benefit here | Rejected as default, not excluded as a future option |

The manifest's `hash_algorithm` field ([01](01-repository-specification.md)
§3) is per-Repository, not per-object — but the format reserves a
hash-algorithm tag alongside every stored hash value
(`{"alg": "blake3", "hex": "..."}`) so that a future migration to a new
algorithm can be introduced without invalidating history: old content
keeps its original algorithm tag and verifies against it forever; new
content can use a newer algorithm once adopted. See
[ADR-0008](../adr/ADR-0008-hash-algorithm-agility.md).

## 2. The Merkle structure

Content hashing already makes every Object revision and Attachment
self-verifying. Two more Merkle layers compose on top:

1. **Commit-level:** a commit's `commit_id` is a hash over its
   `object_changes`, `relationship_changes`, and `event_refs`, each of
   which references content-hashed data. This makes `commit_id` a Merkle
   root over everything that commit touched.
2. **History-level:** because `commit_id` includes `parent_commit_ids`,
   the tip commit hash of a branch is transitively a Merkle root over the
   entire ancestor history reachable from it — identical in structure to
   Git's own commit graph, generalized to engineering objects.

No separate "repository-wide Merkle tree" data structure needs to be
maintained — the commit DAG already is one. `integrity/merkle-roots.json`
([07](07-package-format-proposal.md) §3) simply records, per branch, the
current tip commit hash as a convenient, cached "current state fingerprint,"
rebuildable at any time from the Branch Store.

## 3. Digital signatures

**Recommendation: Ed25519** for commit and package signatures. Rationale:
small key/signature size (practical to store inline in a Commit record),
fast verification (important since verification may run on every `open`),
deterministic signatures (no RNG-quality dependency at signing time,
avoiding a historical class of ECDSA implementation bugs), and wide
existing library support across languages.

- **Author identity** is a Capability-defined concept (human account, service
  account, or AI agent identity) — the Repository only requires that an
  `Identity` reference resolve to a public key for signature verification;
  it does not implement identity/account management itself.
- Signing is **optional per commit** at the Repository core level;
  *requiring* it is a Branch protection policy concern
  ([04-branch-architecture.md](04-branch-architecture.md) §4), not a
  hardcoded rule, consistent with "policy as Capability."

## 4. What integrity verification protects against

| Threat | Mechanism that catches it |
|---|---|
| Bit rot / storage corruption | Content hash mismatch on read |
| Tampering with a past revision's content | Content Hash mismatch on recompute ([02](02-object-storage-architecture.md) §1, §4); the enclosing Commit's hash also changes, cascading through the DAG |
| Tampering with commit history | Commit hash changes, all descendant commit hashes change, signatures (if present) fail |
| Forged authorship | Signature verification fails against claimed Identity's public key |
| Undetected partial/incomplete package | `package_kind` + package-level signature ([07](07-package-format-proposal.md) §5–6) |
| Silent journal loss | Journal entry chaining ([05](05-transaction-journal-architecture.md) §2) detects gaps |

Integrity verification explicitly does **not** protect against a validly
signed, validly chained, but *semantically wrong* engineering change (e.g.
a correctly-authored commit recording an incorrect measurement) — that is
a validation/rules concern layered above the Repository, matching
[03-commit-architecture.md](03-commit-architecture.md) §6.

## 5. Verification levels

`oep verify` ([09](09-cli-specification.md)) supports incremental depth,
since full historical verification of a decades-old Repository can be
expensive:

| Level | Checks | Cost | Required on every `open`? |
|---|---|---|---|
| **0 — Structural** | Manifest readable and version-compatible; journal has no unresolved gaps; every branch's `head_commit` exists in the Commit Store | O(branches + pending journal entries) | Yes |
| **1 — Content** | Recompute content hashes for every Object revision and Attachment reachable from every branch tip (or a specified subset) | O(size of reachable content) | No — configurable, e.g. weekly or on explicit request |
| **2 — History** | Walk the full commit DAG from every branch tip to root; verify parent linkage, recompute every `commit_id`, verify every present signature | O(history length) | No |
| **3 — Semantic** | Re-run validation rules and compare to stored `validation_results` | Delegated entirely to higher-layer Capabilities (Canon); Repository only stores/exposes the comparison hook | Never — not a Repository responsibility |

Levels are cumulative and independently invokable (`oep verify --level 2`),
letting large, long-lived Repositories trade verification cost against
assurance on their own schedule rather than paying full-history
verification cost on every routine `open`.
