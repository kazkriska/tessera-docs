# Project Charter & Architectural Vision
## FRAME: Filesystem Resource Automation & Management Engine

---

## 1. Purpose

This document serves as the foundational charter for **FRAME** (*Filesystem Resource Automation & Management Engine*). It defines the vision, core philosophy, architectural principles, domain terminology, design constraints, and non-goals of the system. Every subsequent specification, component specification, and RFC in the FRAME ecosystem MUST conform to the invariants established in this charter.

---

## 2. Motivation & Problem Statement

Modern automation and AI agent workflows suffer from a fundamental mismatch:
- **Centralized Silos**: Traditional task and workflow engines rely on centralized databases, SaaS applications, or complex orchestrators (e.g., Temporal, Airflow, Jira APIs), making work items coupled to infrastructure.
- **Static File Archives**: Raw filesystems store rich documents, code, images, and data, but directories are passive—they cannot react to change, maintain their own lifecycle, or execute actions autonomously.
- **Agent Handoff Complexity**: When AI agents or human engineers work on modular tasks, passing context, environment variables, dependencies, and execution logs across tool boundaries is fragile and unstructured.

**FRAME** solves this by unifying **data, metadata, state, scripts, and triggers** into self-contained directory bundles (`*.ticket`). These directory bundles are portable across environments, versionable via standard tools (Git), readable by humans and AI agents, and made reactive through a decoupled runtime daemon.

---

## 3. Terminology Glossary

| Term | Definition |
| :--- | :--- |
| **FRAME** | Filesystem Resource Automation & Management Engine—the overall platform and specification suite. |
| **Ticket (`*.ticket`)** | The fundamental resource unit in FRAME. A directory bundle containing metadata, state, logs, working assets, local environment definitions, and a manifest contract. |
| **Manifest (`MANIFEST.yaml`)** | The declarative contract inside a Ticket defining its identity, capabilities (`actions`), event triggers (`hooks`), file watches, and environment dependencies. |
| **Runtime Daemon (`framed`)** | The system daemon/controller process that monitors designated workspaces, maintains the ephemeral event bus, resolves dependencies, and executes hooks/actions. |
| **Workspace** | A root filesystem directory containing one or more `.ticket` bundles monitored by the FRAME runtime. |
| **Hook** | An event-driven subscription defined in a Ticket's manifest that executes a script when a low-level file change or high-level domain event fires. |
| **Action** | A named, callable capability declared in a Ticket's manifest exposed to external CLI callers, subagents, or peer tickets. |
| **Ephemeral Projection / Cache** | The in-memory or local SQLite index maintained by the runtime for fast query resolution. It is 100% rebuildable from disk. |

---

## 4. Architectural Vision & Core Philosophy

### 4.1 The Filesystem as the Universal Source of Truth
In FRAME, there is **no external database** required for core operations. State is not locked inside Postgres, Redis, or a cloud vendor's API. 
- A Ticket can be moved using standard shell commands (`mv`, `cp`, `rsync`).
- A Ticket can be versioned directly inside a Git repository.
- Inspecting a Ticket's state requires only standard CLI tools (`cat`, `jq`, `yq`).

If the FRAME runtime daemon crashes, is killed, or is deleted, **zero state is lost**. Upon restart, the runtime daemon re-scans the filesystem, reads all `MANIFEST.yaml` and `state.json` files, rebuilds its internal lookup graph, and resumes monitoring.

### 4.2 "Ticket" as the Fundamental Resource Abstraction
In traditional software, a "ticket" implies a issue tracker item. In FRAME, **Ticket is a resource-agnostic abstraction**, analogous to how Linux treats "everything as a file" or Kubernetes treats "everything as a Custom Resource (CRD)."

A Ticket represents any unit of work, memory, skill, pipeline step, or agent task that possesses:
1. **Identity**: Unique identifier and scope.
2. **State & Lifecycle**: Known status (`created`, `ready`, `running`, `blocked`, `completed`, `failed`).
3. **Data & Context**: Input documents, output assets, metadata.
4. **Behavior**: Callable actions and reactive event hooks.
5. **Relationships**: Links to parent tickets, sub-tickets, dependencies, or sibling resources.

Whether a Ticket represents a user bug report, a data transformation pipeline, an AI subagent context, or an automated deployment workflow is simply a function of its `kind`, `metadata`, and `MANIFEST.yaml` hooks.

---

## 5. System Invariants & Guiding Principles

Every subsystem built for FRAME must adhere to these five non-negotiable invariants:

### Invariant 1: Single Source of Truth Disk Locality
> *State resides in the directory bundle. Any external database or index is an ephemeral read-side cache.*

### Invariant 2: Declarative Contract / Imperative Handler Separation
> *The `MANIFEST.yaml` declares WHAT events a Ticket listens to and WHAT actions it exposes. Handlers in `scripts/` perform HOW execution occurs.*

### Invariant 3: Rebuildability
> *Deleting the runtime cache or index MUST NOT cause data corruption or loss of domain state. Cold-boot discovery must fully restore system graph state.*

### Invariant 4: Subprocess Isolation & Determinism
> *Hooks and actions execute as isolated subprocesses with explicit timeouts, controlled environment variable merging, and sandboxed file paths.*

### Invariant 5: Idempotency & Debouncing
> *Filesystem events are noisy. The runtime MUST debounce rapid file writes and provide event correlation IDs to ensure handlers operate idempotently.*

---

## 6. Design Constraints & Non-Goals

### 6.1 Constraints
- **POSIX Filesystem Compatibility**: Must run efficiently on Linux and macOS POSIX filesystems utilizing native event notification primitives (`inotify`, `kqueue`, `FSEvents`).
- **Zero Heavy Runtime Dependencies**: The core runtime daemon must run using standard Python 3.12+ (or static binary in future releases) without requiring external services like Docker, Redis, or PostgreSQL.

### 6.2 Non-Goals
- **FRAME is NOT a GUI issue tracker**: While a web dashboard can be built on top of FRAME's event bus or CLI, FRAME itself is an infrastructure runtime.
- **FRAME is NOT an in-memory message broker**: FRAME does not replace high-throughput event streaming systems like Kafka. Event rates are tied to filesystem operational cadences (seconds/milliseconds, not millions per second).
- **FRAME does NOT replace Git**: FRAME orchestrates execution and state transitions; it does not implement custom version control algorithms.

---

## 7. Naming Rationale

### Why "FRAME"?
The name **FRAME** (*Filesystem Resource Automation & Management Engine*) was chosen because:
- **Structural**: It provides the architectural "framing" inside which files, scripts, and AI agents operate.
- **Predictable**: Reflects the three pillars of the system: **Filesystem-Native**, **Resource-Agnostic**, **Automation-Driven**.
- **Executable**: Simple, memorable CLI command (`frame watch`, `frame run`, `frame inspect`).

---

## 8. Rationale Summary Matrix

| Decision | WHAT | WHY | HOW |
| :--- | :--- | :--- | :--- |
| **Filesystem as Source of Truth** | All state, logs, and manifests live in `.ticket/` directories. | Eliminates database locks, enables git versioning, simplifies relocation and backup. | Runtime watches files via `inotify`/`watchdog` and updates local `state.json` atomically. |
| **Generic Resource Model** | Treat all workflows, tasks, skills, and agents as "Tickets". | Avoids multiplying entity types; provides a single unified lifecycle contract. | Discovered via `kind` in `MANIFEST.yaml` and structured `metadata.json`. |
| **Disposable Ephemeral Index** | Maintain an in-memory/SQLite projection for fast queries. | Speeds up dependency graphs and CLI queries without sacrificing filesystem truth. | Scans workspace root on startup; invalidates index entry on disk deletion/modification. |
| **Polyglot Execution** | Support Python (`uv`), Bash, and Node scripts in `scripts/`. | Developers and AI agents can write handlers in whichever language fits the task best. | Runtime spawns subprocesses passing event payloads via STDIN/JSON and capturing STDOUT to `activity.jsonl`. |
