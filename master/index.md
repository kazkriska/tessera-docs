# Tessera — Master Specification

**Status:** Stable (Phase 1 — Master)
**Derived from:** `foundation/charter.md` (Phase 0)
**Superseded by:** `rfcs/` (Phase 2) for versioned, independently-evolving specs, which reference this document.

This is the canonical "book." It is the authoritative design reference from which the framework can
be implemented without reverse-engineering decisions. Once stable, it is the basis for the RFC suite
(Phase 2), which extracts each major Part into a versioned RFC.

---

## Conventions used throughout

- **WHAT / WHY / HOW** framing accompanies every major component: *what it does*, *why it is designed
  this way*, *how it interacts with the rest of the framework*.
- **Invariants (I-n)** reference `foundation/charter.md` §6.
- **Tickets** are referenced by immutable `id` (e.g. `HQ_BR-001`).
- **Paths** are shown relative to `FrameworkRoot/` unless absolute is stated.

## Section template

Each Part follows this predictable structure so the document is navigable:

1. Purpose
2. Motivation
3. Responsibilities
4. Design
5. Directory Layout
6. Lifecycle
7. Interaction (with other components)
8. Failure Modes
9. Security
10. Future Extensions
11. Examples
12. Rationale

## Directory layout (authoritative)

```
FrameworkRoot/
├── Tickets/                         # = TicketRepository (canonical, Git-tracked)
│   ├── .ticket-runtime/            # disposable runtime state
│   │   ├── registry.db             # derived index of tickets
│   │   ├── locks/                  # per-ticket / per-action lock files
│   │   ├── cache/                  # manifest hashes, dep graphs, renders, embeddings
│   │   ├── logs/                   # runtime logs (distinct from ticket activity.jsonl)
│   │   ├── runtime.sock            # IPC + singleton enforcement
│   │   ├── config.yaml             # workspace/runtime settings
│   │   ├── plugins/                # installed runners/providers
│   │   └── tmp/                    # safe-to-delete scratch
│   ├── HQ_BR-001.ticket/           # a Ticket (source of truth)
│   │   ├── MANIFEST.yaml           # declarative behavior
│   │   ├── metadata.json           # structured metadata
│   │   ├── state.json              # mutable runtime state
│   │   ├── activity.jsonl          # append-only history
│   │   ├── task/                   # work artifacts
│   │   │   └── assets/
│   │   ├── scripts/                # reusable scripts (invoked by hooks/actions)
│   │   ├── hooks/                  # event-reactive code (optional; manifest is source)
│   │   └── .env                    # execution environment (overrides global)
│   └── HQ_BR-002.ticket/
├── Skills/                         # skill catalogue + skill dirs (own resource type later)
│   ├── Skill-Catalogue.json
│   ├── abcd/
│   ├── xyz/
│   └── lmn/
├── lib/
│   └── ticket-management/          # the runtime implementation
│       ├── runtime/
│       │   ├── watcher.py
│       │   ├── dispatcher.py
│       │   ├── manifest.py
│       │   ├── executor.py
│       │   ├── state.py
│       │   └── env.py
│       ├── plugins/
│       │   ├── python_runner.py
│       │   ├── bash_runner.py
│       │   └── node_runner.py
│       ├── cli.py
│       └── pyproject.toml
└── README.md
```

> Note: Global `.env` lives at `FrameworkRoot/Tickets/.env` (global defaults). Per-Ticket `.env`
> overrides it. Resolution order (later overrides earlier): Global → Ticket `.env`.

## Scope of the Master

| Part | Subject |
| --- | --- |
| I | Vision |
| II | Architecture (overall pipeline & component map) |
| III | Runtime Internals |
| IV | Ticket Model |
| V | Manifest Specification (v1) |
| VI | Lifecycle & State Machine |
| VII | Event Bus |
| VIII | Scheduler |
| IX | Permissions & Security |
| X | Command-Line Interface |
| XI | SDK Architecture |
| XII | Implementation Roadmap |

## Reading order recommendation

Charter → Master index → Part II (Architecture) → Part IV (Ticket Model) → Part V (Manifest) →
Part VI (Lifecycle) → Part III/VII/VIII (Runtime, Event Bus, Scheduler) → Part IX–XI (Permissions,
CLI, SDK) → Part XII (Roadmap). Parts III/VII/VIII may be read together since they form the
execution pipeline.

## Document map

| Document | Subject | Phase |
| --- | --- | --- |
| `foundation/charter.md` | Charter (Phase 0) | Phase 0 |
| `master/index.md` | Master Specification (this document) | Phase 1 |
| `master/part-i-vision.md` … `master/part-xii-roadmap.md` | Master Parts I–XII | Phase 1 |
| `rfcs/README.md` | RFC Suite Index | Phase 2 — RFCs |
| `rfcs/rfc-0000-philosophy.md` | RFC-0000 — Philosophy & Guiding Principles | Phase 2 — RFCs |
| `rfcs/rfc-0001-architecture.md` | RFC-0001 — Architecture & Component Pipeline | Phase 2 — RFCs |
| `rfcs/rfc-0002-ticket-model.md` | RFC-0002 — Ticket Model & Resource Abstraction | Phase 2 — RFCs |
| `rfcs/rfc-0003-manifest.md` | RFC-0003 — Manifest Specification (v1) | Phase 2 — RFCs |
| `rfcs/rfc-0004-runtime.md` | RFC-0004 — Runtime Internals | Phase 2 — RFCs |
| `rfcs/rfc-0005-event-bus.md` | RFC-0005 — Event Bus | Phase 2 — RFCs |
| `rfcs/rfc-0006-scheduler.md` | RFC-0006 — Scheduler, Queues & Locking | Phase 2 — RFCs |
| `rfcs/rfc-0007-lifecycle.md` | RFC-0007 — Ticket Lifecycle & State Machine | Phase 2 — RFCs |
| `rfcs/rfc-0008-permissions.md` | RFC-0008 — Permissions & Security Model | Phase 2 — RFCs |
| `rfcs/rfc-0009-cli.md` | RFC-0009 — Command-Line Interface | Phase 2 — RFCs |
| `rfcs/rfc-0010-sdk.md` | RFC-0010 — SDK Architecture | Phase 2 — RFCs |
| `rfcs/rfc-0011-extension.md` | RFC-0011 — Extension System & Future Runtimes | Phase 2 — RFCs |
| `rfcs/rfc-0012-roadmap.md` | RFC-0012 — Implementation Roadmap | Phase 2 — RFCs |
| `rfcs/rfc-0013-distribution.md` | RFC-0013 — Distribution & Installation | Phase 2 — RFCs |

> **RFC-0013 is the authoritative distribution & installation spec** for Tessera v1 (tarball-based
> installer, `curl … | bash` bootstrap, user-scoped systemd install). Where any other document
> disagrees on distribution/installation behavior, RFC-0013 takes precedence.

---

> **REVISION — Layout rename (Rev B · 2026-08-08).**
> *Non-destructive: prior text unchanged, remains canonical. The package tree moved to a src/ layout: `lib/ticket-management/` → `src/tessera_runtime/`; `tessera/` (SDK) → `src/tessera_sdk/`. Imports: `from tessera import …` → `from tessera_sdk import …`; `ticket_management.cli:main` → `tessera_sdk.cli:main`. Status: Appended.*

### R.B.1 — Affected references in this document

The rows below map canonical text above (left) to its Rev B equivalent (right). The canonical text is **not** rewritten; read it through this table.

| Location (canonical text) | As written (Rev A, canonical) | Rev B equivalent |
|---|---|---|
| § Conventions / repository tree | `lib/` | `src/` |
| § Repository tree | `lib/ticket-management/` — the runtime implementation | `src/tessera_runtime/` — the runtime implementation |
| § Repository tree | SDK package `tessera/` | `src/tessera_sdk/` |
| § Repository tree | `lib/ticket-management/pyproject.toml` | `pyproject.toml` at FrameworkRoot, packaging `src/tessera_runtime` + `src/tessera_sdk` |

No canonical sentence above is amended by this block; tooling that resolves paths or imports MUST apply the mapping table. Rev A text remains the authority on *behaviour*; Rev B is the authority on *location*.

---

## Document Revisions

| Rev | Date | Note |
| --- | --- | --- |
| A | (initial) | Master Specification published (Phase 1). |
| B | 2026-08-08 | Layout rename: `lib/ticket-management/` → `src/tessera_runtime/`; `tessera/` → `src/tessera_sdk/`. |
| C | 2026-08-08 | Added RFC-0013 — Distribution & Installation as the **authoritative distribution spec** (tarball-based installer, `curl … | bash` bootstrap, user-scoped systemd install); added the Document map table referencing it. |
