# Repository State Machine

Status: Draft v0.1 (Sprint 0.5) · Formal session-level state machine, operating within the Active/Synchronized/Restored states of [23-repository-lifecycle.md](23-repository-lifecycle.md).

## 1. States

```
Closed ──open()──▶ Opening ──validated──▶ Open/Idle
                       │                      │
                  validation                  │ begin_transaction()
                  fails                        ▼
                       │                 Transacting ──commit()──▶ Open/Idle
                       ▼                       │
                  FailedOpen              rollback()/failure
                       │                        │
                  (requires repair,              ▼
                   12-repository-internals-  RolledBack ──▶ Open/Idle
                   lifecycle-and-maintenance.md §3-6)
                       │
                       ▼
                    Repaired ──▶ Opening (retry)

Open/Idle ──sync()──▶ Syncing ──complete──▶ Open/Idle
                          │
                     sync conflict
                          ▼
                  ConflictPending ──resolved──▶ Open/Idle
                                                    │
Open/Idle ──close()──▶ Closing ──▶ Closed              │
                                                    │
Open/Idle ──maintenance ops──▶ Maintaining ──▶ Open/Idle
  (optimize/repack/gc, 20-package-maintenance.md,
   12-repository-internals-lifecycle-and-maintenance.md)

Any state ──unrecoverable I/O or integrity failure──▶ Faulted (terminal
                                                        for this session;
                                                        requires a new
                                                        open() attempt,
                                                        which re-enters
                                                        Opening)
```

## 2. State definitions

| State | Description |
|---|---|
| **Closed** | No active session. Initial and normal-exit state. |
| **Opening** | `open()` called; manifest read, `format_version` checked, journal replay in progress ([01-repository-specification.md](01-repository-specification.md) §4.2, [05-transaction-journal-architecture.md](05-transaction-journal-architecture.md) §3). |
| **Open/Idle** | Validated and ready; no transaction currently in flight. The steady state for reads and for issuing new transactions. |
| **Transacting** | A Transaction ([25-repository-transactions.md](25-repository-transactions.md)) is in flight — one or more journal entries written, not yet committed or aborted. |
| **RolledBack** | A Transaction was explicitly rolled back or aborted after failure; store state has been confirmed unchanged (or restored) before returning to Open/Idle. |
| **FailedOpen** | `open()` could not complete validation (integrity failure, unreadable Header, unsupported major format version). Does not silently degrade to Open/Idle. |
| **Repaired** | A repair Operation ([12-repository-internals-lifecycle-and-maintenance.md](12-repository-internals-lifecycle-and-maintenance.md) §3, [20-package-maintenance.md](20-package-maintenance.md) §6) completed against a FailedOpen Repository; re-attempts Opening rather than assuming success. |
| **Syncing** | A synchronization exchange with one or more peers is in progress ([26-synchronization-philosophy.md](26-synchronization-philosophy.md)). |
| **ConflictPending** | Sync detected a conflict requiring resolution ([26](26-synchronization-philosophy.md) §5, [27-multi-user-behavior.md](27-multi-user-behavior.md)) before the Repository can return to a fully converged Open/Idle state. Reads remain available; new local commits MAY be restricted by policy while pending, per Capability advertisement ([22-repository-capability-model.md](22-repository-capability-model.md)). |
| **Maintaining** | An optimize/repack/GC/upgrade Operation is running ([12](12-repository-internals-lifecycle-and-maintenance.md), [20](20-package-maintenance.md)). Concurrent transactions are queued or rejected depending on the specific maintenance Operation's isolation requirements (§3). |
| **Closing** | Graceful shutdown: any in-flight Transaction is required to have reached Committed or RolledBack first; Closing does not itself decide that — it is only reachable from Open/Idle. |
| **Faulted** | Unrecoverable failure detected mid-session (e.g. underlying Storage Provider connection lost, disk I/O error). Terminal for the current session; recovery requires a fresh `open()` call, re-entering Opening (which will itself run journal replay/crash recovery). |

## 3. Concurrency between states and maintenance

Maintenance Operations (§ Maintaining) vary in whether they require
exclusive access:

- **Repack/defragment** ([20-package-maintenance.md](20-package-maintenance.md)
  §2-3) write an entirely new file and atomically swap — readers against
  the *old* file continue unaffected until the swap; this can run
  concurrently with read-only Open/Idle sessions.
- **GC** ([12-repository-internals-lifecycle-and-maintenance.md](12-repository-internals-lifecycle-and-maintenance.md)
  §1) requires that no Transaction is relying on a soon-to-be-removed
  unreachable commit's content becoming reachable again — implemented as
  a brief exclusive phase (compute the reachable set, quarantine
  candidates) rather than a long exclusive lock over the whole run.
- **Format upgrade** ([12](12-repository-internals-lifecycle-and-maintenance.md)
  §4) requires exclusive access for its full duration, since it changes
  the canonical serialization rules every concurrent reader/writer would
  otherwise assume.

## 4. Failure states are first-class, not error returns

FailedOpen, RolledBack, and Faulted are named states, not just error codes
returned from a function call — this is deliberate: tooling built against
this state machine (a GUI, a monitoring dashboard, a CI job) can display
"this Repository is in FailedOpen" as a durable, inspectable condition,
and recovery tooling can target a specific named state rather than
pattern-matching on error messages.
