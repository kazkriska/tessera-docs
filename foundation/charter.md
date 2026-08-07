# Tessera — Project Charter & Architecture Vision

**Status:** Ratified (Phase 0 — Foundation)
**Audience:** Engineers, contributors, and future subsystem designers
**Purpose:** Establish vision, philosophy, terminology, principles, constraints, non-goals, and
core abstractions. Every later Part and RFC is derived from this document and must not contradict it.

---

## 1. Purpose

Tessera exists because the user wanted a **self-contained directory** — carrying files, images,
documents, scripts, metadata, and configuration — that can **react to changes in itself and in
peer directories**, executing automation that mutates its own state or the state of other
directories. The naive assumption was that Python `venv`/`uv` provided this behavior. It does
not: a virtual environment only isolates dependencies and does nothing to watch files, emit
events, or execute code on change.

The realized design is an **event-driven, filesystem-native automation framework** in which each
directory is a declarative resource (a *Ticket*) and a single runtime interprets those resources.
This Charter records that design as the project's canonical intent.

## 2. Motivation

### Why not a virtual environment?
A `venv` or `uv` environment is a Python interpreter plus isolated packages. It has no watcher,
no trigger, no lifecycle, and no relationship model. Treating it as the automation layer conflates
*dependency isolation* with *behavior*. The two are orthogonal; the runtime uses `uv` only to
provide the Python execution environment for hooks that need packages.

### Why not bash-only?
For tiny systems, `inotifywait` + a `case` statement works. But the moment the system handles
dependency graphs, parent/child relationships, state transitions, agents, retries, asynchronous
tasks, and JSON manipulation, a general-purpose runtime becomes far easier to maintain. Bash stays
useful for orchestration (git, rsync, mkdir) but not as the engine.

### Why a runtime at all?
The Ticket should be a **declarative package**; the runtime is the **generic interpreter**. This
separation lets hundreds or thousands of Tickets share one engine, keeps Tickets portable, and lets
the runtime evolve (e.g. a future Rust implementation) without changing the Ticket format.

## 3. Vision

A framework in which **the filesystem is the program**. Each resource is a self-describing,
portable tile; the runtime gives those tiles "life" by watching them, translating changes into
events, and executing declared automation. Today the primary resource is the Ticket; tomorrow it
may be Skills, Workflows, Memories, Prompts, Agents, and Datasets — all of which decompose into
atomic units that are themselves Tickets.

The runtime is not a ticket system. It is a **control plane for filesystem-native resources**, of
which the Ticket runtime is the first concrete instance.

## 4. Philosophy

1. **Filesystem is the source of truth.** Tickets are the canonical state. The runtime is a
   derived view.
2. **Tickets are portable and self-contained.** A Ticket directory can be zipped, moved to another
   machine, committed to Git, or sent to a colleague unchanged.
3. **Runtime is stateless and rebuildable.** All runtime state is derivable from the filesystem by
   rescanning. Delete `.ticket-runtime/` and the runtime recovers.
4. **Manifests are declarative.** The manifest describes *what* should happen; code implements *how*.
   No conditional logic belongs in the manifest.
5. **Runtime state is disposable.** Registry, caches, locks, logs, and sockets are implementation
   details, safe to delete.
6. **Everything is event-driven.** Filesystem changes become events; events drive automation.
   Hooks never call each other directly — they emit events.
7. **Extensibility over hardcoding.** Behavior is discovered from manifests and pluggable runners,
   not from directory conventions or baked-in paths.
8. **Language-agnostic hook execution.** A hook may be Python, Bash, Node, or any executable the
   runtime has a runner for.
9. **Explicit configuration over convention.** The manifest is the single source of behavior. The
   runtime does not infer behavior from file names like `scripts/on_metadata_changed.py`.
10. **Single runtime per workspace (TicketRepository).** Multiple runtimes watching the same
    repository create duplicate execution and corruption. A PID file / socket enforces singularity.

## 5. Terminology

| Term | Definition |
| --- | --- |
| **FrameworkRoot** | Top-level directory of a Tessera deployment (e.g. `~/FrameworkRoot/`). Contains the repository, skills, and `lib/`. |
| **TicketRepository** | Directory holding all Ticket directories (`./Tickets` in earlier drafts, renamed to avoid overloading "workspace"). Canonical, Git-tracked, human/agent-editable. |
| **Ticket** | The fundamental resource abstraction. A self-contained directory `HQ_BR-001.ticket/` with manifest, metadata, state, history, config, and automation. |
| **Workspace** *(reserved)* | An **external** directory where work is actually performed by a human or AI agent. Declared in `.env` or `metadata.json`. NOT where Tickets are stored. |
| **Manifest** | `MANIFEST.yaml` inside a Ticket: a purely declarative description of lifecycle hooks, actions, permissions, relationships, and execution metadata. |
| **metadata.json** | Structured, machine-readable metadata about the Ticket (owner, id, title, type, relationships). |
| **state.json** | Mutable runtime state (current lifecycle state, assignee, locks held, computed values). |
| **activity.jsonl** | Append-only, one JSON object per line, recording the Ticket's lifecycle and event history. Inside the Ticket. |
| **.env** | Execution environment (API keys, paths, PYTHONPATH). Per-Ticket `.env` overrides global. |
| **.ticket-runtime/** | Hidden directory *inside the TicketRepository* holding disposable runtime state (registry.db, locks/, cache/, logs/, runtime.sock, config.yaml, plugins/, tmp/). |
| **Runtime** | The long-running Python 3.12+ daemon that watches the repository, builds the registry, and executes automation. |
| **Event Bus** | In-process publish/subscribe channel carrying domain events (notifications). |
| **Scheduler** | Component between Event Bus and Dispatcher that owns queues, priorities, retries, delays, rate limits, locks, and dependency ordering. |
| **Dispatcher** | Selects the appropriate runner/executor for a given hook or action. |
| **Executor / Runner** | Actually runs a hook/action in its language (python, bash, node, …). |
| **Hook** | Code that *reacts* to an event. Declared under `hooks:` in the manifest. |
| **Action** | A named, callable capability. Invocable by users, hooks (via event), agents, or CLI/SDK. |
| **Registry** | Derived SQLite index of discovered Tickets (id → path, parent, state, manifest version, last scanned). |
| **Lifecycle** | Fixed set of states (Created, Initialized, Ready, Running, Blocked, Delegated, Completed, Archived) with flexible but constrained transitions. |

## 6. Guiding Principles (design constraints)

These are **invariants** — properties that must remain true in any conforming implementation:

- **I-1.** The contents of `TicketRepository/` are never modified by the runtime except through
  explicitly declared, locked state mutations (state.json, activity.jsonl, and files a hook/action
  writes under its granted permissions).
- **I-2.** Deleting `.ticket-runtime/` never loses Ticket data and never corrupts a Ticket.
- **I-3.** Every Ticket has exactly one immutable `id`. Titles may change; IDs do not.
- **I-4.** Events are notifications. Anything requiring a return value uses an action invocation,
  not event payloads.
- **I-5.** A hook may only trigger further automation by emitting an event, never by directly
  invoking another hook or action's code path.
- **I-6.** Exactly one runtime process owns a given TicketRepository at a time.
- **I-7.** The manifest is data. Conditional/looping logic is forbidden in the manifest; it lives
  in hooks/actions.
- **I-8.** Locks are acquired only around state mutations, at ticket and action granularity.
- **I-9.** Rescanning the repository from scratch always reconstructs an equivalent registry.

## 7. Non-Goals (explicitly out of scope for v1)

- **Agent orchestration as a first-class contract.** Delegation/handoff/orchestration are
  *conceptualized* but implemented merely as another action provider in v1. The full agent contract
  is deferred until the wider framework (not just ticket-management) is specified.
- **AI/ML built in.** AI is kept intentionally abstract behind providers (local models, OpenRouter,
  MCP, embeddings, memory). No AI provider is mandated in v1.
- **Full plugin marketplace.** Only the runners/modules strictly needed now are defined. A general
  plugin system is a future extension point.
- **Compiled/distributable Ticket packages.** The earlier idea of compiling a Ticket into a single
  XML/binary artifact is deferred; the directory remains the source of truth (Option A).
- **Windows / macOS support.** Linux-only for v1 (inotify-based watching).
- **Distributed execution across machines.** v1 supports multiple hooks per ticket and prepares for
  later distribution, but does not implement cross-node scheduling.

## 8. Core Abstractions

### 8.1 The Ticket
A Ticket is the unit of automation. It bundles:
- **Data** (`metadata.json`, `task/`, assets)
- **Configuration** (`MANIFEST.yaml`, `.env`)
- **State** (`state.json`)
- **Behavior** (`scripts/`, `hooks/`)
- **History** (`activity.jsonl`)

The runtime treats a Ticket uniformly regardless of what it *represents*. A feature request, a
memory entry, and a skill are distinguished only by `metadata.json` and by the manifest's declared
behavior.

### 8.2 The Runtime
A single Python process per TicketRepository responsible for:
loading manifests, watching the filesystem, resolving parent/child and graph relationships, merging
configuration, dispatching events, executing hooks/actions, tracking state, logging activity, and
handling retries/failures.

### 8.3 The Event Bus
Domain events (`metadata.updated`, `ticket.assigned`, `parent.changed`, `status.completed`,
`workspace.initialized`, …) flow through an in-process bus. Hooks subscribe; actions are invoked.
The bus decouples producers from consumers.

### 8.4 The Scheduler
Owns execution policy: queues, worker pools, priorities, retries, delays, rate limits, lock
acquisition, and dependency ordering. Without it, scheduling logic leaks into the Dispatcher.

### 8.5 The Executor / Runner
Runs a declared hook or action in its language. Runners are pluggable (python, bash, node, …). The
runtime is language-agnostic at the manifest level and language-aware only at the runner level.

### 8.6 The Repository
A directory of Tickets. Canonical, editable, portable. The runtime indexes it but does not own it.

## 9. Why the design is the way it is (selected rationale)

- **One watcher, not per-Ticket watchers.** A single `inotify` watch over the repository is cheaper
  (CPU, FDs) and far easier to debug than N watchers. The runtime maps a changed file back to its
  owning Ticket.
- **Registry is derived, not authoritative.** Rescan-rebuildable state simplifies recovery, makes
  Tickets trivially portable, and removes a second source of truth that could drift from disk.
- **Scheduler as a distinct stage.** Keeps the Dispatcher free of queueing/retry/lock logic, so each
  component has one responsibility (matching the long-term controller pipeline:
  Filesystem → Discovery → Registry → Watcher → Event Bus → Scheduler → Dispatcher → Executor → Ticket).
- **Graph relationships, not just parent/child.** A ticket may `depends_on`, `blocks`, `duplicates`,
  `references`, `relates_to`, `spawned_from`, or `delegated_to` others. Parent/child is one relationship
  type among many.
- **Immutable IDs + mutable titles.** Enables stable references and rename safety; relationships and
  logs key off the ID.
- **activity.jsonl over plain text.** Machine-readable append-only history supports tooling,
  auditing, and replay without losing human readability.
- **`apiVersion: ticket/v1` + `kind: Ticket`.** Kubernetes-style versioning lets the format evolve
  without breaking existing Tickets.

## 10. Evolution expectations

The Charter is written so the framework can grow into a family of runtimes:
`ticket-runtime`, `workflow-runtime`, `skill-runtime`, `agent-runtime`, sharing an Event Bus,
Scheduler, Runner/Permission/Registry substrate. The Ticket runtime is the first. Future
repositories (SkillRepository, WorkflowRepository, MemoryRepository) follow the same shape.

## 11. Relationship to later documents

- **Master Specification (Phase 1)** expands each abstraction into a Part with the standard section
  template (Purpose, Motivation, Responsibilities, Design, Layout, Lifecycle, Interaction, Failure
  Modes, Security, Future Extensions, Examples, Rationale).
- **RFC Suite (Phase 2)** is *derived* from the stable Master; RFCs version independently and
  reference the Master rather than duplicating it.

This Charter may only be amended by explicit, versioned revision. It is the root of all authority.
