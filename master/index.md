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
