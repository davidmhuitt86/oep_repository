# Repository Lifecycle

Status: Draft v0.1 (Sprint 0.5) · Defines the lifecycle states a Repository instance passes through over its (potentially decades-long) existence.

## 1. Lifecycle states

The directive's suggested linear lifecycle
(Created → Initialized → Active → Synchronized → Archived → Restored →
Deprecated → Migrated → Retired) is refined below into a graph rather than
a strict line, because several of these states are not mutually exclusive
phases a Repository passes through once — a Repository is routinely
Active *and* periodically Synchronized throughout its life, and Archived
Repositories are sometimes Restored back to Active more than once.

```
                    ┌─────────────┐
                    │   Created   │   (oep init, before first commit)
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │ Initialized │   (manifest + empty branch exist)
                    └──────┬──────┘
                           │ first commit
                    ┌──────▼──────┐
              ┌────▶│   Active    │◀────┐
              │     └──┬───┬───┬──┘     │
              │  sync  │   │   │ restore│
              │        ▼   │   │        │
              │  ┌─────────┐│  │  ┌──────────┐
              └──│Synchronized│ └─▶│ Restored │
                 └─────────┘       └────┬─────┘
                    │                    │
                    │ seal               │ (rejoins Active)
                    ▼                    │
              ┌───────────┐              │
              │  Archived │──────────────┘
              └─────┬─────┘
                     │ superseded by a newer format/successor project
                     ▼
              ┌─────────────┐        ┌───────────┐
              │ Deprecated  │───────▶│ Migrated  │
              └─────┬───────┘        └─────┬─────┘
                     │                       │
                     └───────────┬───────────┘
                                 ▼
                          ┌───────────┐
                          │  Retired  │   (terminal)
                          └───────────┘
```

## 2. State definitions

| State | Meaning | Entry condition | Capability implications |
|---|---|---|---|
| **Created** | `oep init` has run; manifest exists; no commits yet ("unborn" branch, [01-repository-specification.md](01-repository-specification.md) §4.1) | `init` | No commit-dependent capability is meaningful yet |
| **Initialized** | Distinguishes "just created" from "has at least been opened and validated once" — mostly a bookkeeping distinction useful for tooling/onboarding flows, not a structurally different state from Created | First successful `open` | Same as Created |
| **Active** | Normal operating state: commits, branches, relationships being created and modified | First commit | All capabilities the Repository advertises apply normally |
| **Synchronized** | Not a resting state — a transient/recurring condition during and immediately after a sync exchange with one or more peers ([26-synchronization-philosophy.md](26-synchronization-philosophy.md)) | Sync operation initiated | Requires `oep.repo.Synchronization` |
| **Archived** | Deliberately sealed: `oep.repo.ImmutableArchiveMode` capability now advertised as `Supported`; no further mutation accepted without first transitioning back to Active via Restore | Explicit `oep archive` Operation | Mutation-dependent capabilities report `Unsupported` while archived |
| **Restored** | An Archived Repository has been explicitly reopened for continued work; transitional state that immediately rejoins Active | Explicit `oep restore` Operation | Reverts to whatever the Repository advertised before archiving |
| **Deprecated** | Still readable, explicitly marked as no longer the canonical/maintained instance of this engineering history (e.g. superseded by a migrated copy) | Explicit marking, does not require Archived first | Advertises `oep.repo.ReadOnlyMode` by convention, not by hard requirement |
| **Migrated** | Content has been carried forward into a new Repository under a newer `format_version` or successor tooling; this instance becomes a historical source, not a live one | Completion of a migration Operation ([12-repository-internals-lifecycle-and-maintenance.md](12-repository-internals-lifecycle-and-maintenance.md) §4) | Points to the successor Repository's identity in its manifest metadata |
| **Retired** | Terminal. Kept only for legal/historical retention if at all; no tooling is expected to actively maintain it | Organizational decision, no format-level trigger | No capabilities assumed |

## 3. Why a graph, not a line

The directive's linear ordering implies each state is visited once, in
order. In practice: a Repository cycles through Active ↔ Synchronized many
times per day, and Archived ↔ Restored potentially many times over a
Repository's life (e.g. a completed OEM Variant branch archived for
years, then restored when that product line resumes). Modeling this as a
strict line would force either fictional re-entry into "Active" as if it
were a fresh state each time, or an ad hoc exception to the model — a
graph avoids both by making the actually-repeatable transitions (sync,
archive/restore) explicit cycles rather than one-way steps.

## 4. Relationship to the Repository State Machine

This lifecycle describes the Repository's *macro* life story — states that
can each last from seconds (Synchronized) to decades (Archived). The
moment-to-moment mechanics of any single Active-state session (opening,
transacting, closing, handling failures) are specified separately in
[24-repository-state-machine.md](24-repository-state-machine.md), which
operates entirely *within* the Active/Synchronized/Restored states here —
the state machine does not itself model Archived/Deprecated/Migrated/
Retired, which are lifecycle-level, not session-level, concerns.
