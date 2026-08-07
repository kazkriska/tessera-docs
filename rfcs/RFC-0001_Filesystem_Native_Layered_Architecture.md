# RFC-0001: Filesystem-Native Layered Architecture

- **RFC Number:** RFC-0001
- **Title:** Layered System Topology and Workspace Discovery
- **Status:** Draft / Active
- **Author:** FRAME Core Architecture Team
- **Created:** 2026-08-07
- **Replaces:** None
- **Category:** Standards Track / Core

---

## 1. Abstract

This RFC specifies the six-layer system architecture of the FRAME Engine (`framed`), detailing the separation of concerns across workspace discovery, file event monitoring, domain event translation, priority queuing, and polyglot execution.

---

## 2. Six-Layer Architecture

```
Layer 1: Filesystem Workspace          [*.ticket Directory Bundles, .env]
Layer 2: Discovery & Registry          [Workspace Scanner, Ephemeral frame.db]
Layer 3: Event Engine & File Watcher   [inotify / watchdog, Debouncer, Event Bus]
Layer 4: Queue & Scheduler             [Priority Task Queue, Concurrency Control]
Layer 5: Dispatcher & Sandboxing       [Permission Checker, Path Jail, Env Merger]
Layer 6: Polyglot Script Runners       [Python/uv Runner, POSIX Shell Runner]
```

---

## 3. Layer Specifications

### 3.1 Layer 1 — Filesystem Workspace
Stores self-contained Ticket directory bundles (`*.ticket/`). Contains no shared memory or active daemon processes; communication occurs strictly via disk file changes and domain events.

### 3.2 Layer 2 — Discovery & Registry Engine
Recursively scans workspace roots for valid `MANIFEST.yaml` files. Maintains an ephemeral SQLite database (`frame.db` at `~/.cache/frame/frame.db`) indexing ticket IDs, filesystem paths, statuses, and manifest SHA-256 hashes.

### 3.3 Layer 3 — Event Engine & File Watcher
Monitors OS-level file events (`IN_MODIFY`, `IN_CREATE`, `IN_DELETE`) using `inotify` or `watchdog`. Applies a $300\text{ms}$ sliding-window debouncing filter to merge rapid edits into single high-level domain events (`ticket.metadata.updated`, `ticket.state.changed`).

### 3.4 Layer 4 — Priority Task Queue & Scheduler
Manages execution ordering via a thread-safe priority queue. Assigns priority levels (0: Emergency, 1: Direct CLI/User, 2: File Hooks, 3: Background Maintenance) and caps concurrent subprocess workers to prevent CPU saturation.

### 3.5 Layer 5 — Dispatcher & Security Boundary
Validates execution paths to prevent directory traversal outside workspace boundaries. Merges environment variables in order: System Base $\rightarrow$ Workspace `.env` $\rightarrow$ Ticket `.env` $\rightarrow$ Manifest Payload.

### 3.6 Layer 6 — Polyglot Runners
Invokes script handlers via isolated subprocesses. Supports Python execution via `uv` or local `.venv` environments, POSIX Bash scripts (`set -euo pipefail`), and custom runners.

---

## 4. References & Related RFCs

- [RFC-0000: Philosophy and Guiding Principles](file:///home/kaz/NewProjects/UnnamedFileSystemAutomationProject/docs/rfcs/RFC-0000_FRAME_Philosophy_and_Guiding_Principles.md)
- [RFC-0004: Runtime Engine Event Bus and Debouncing](file:///home/kaz/NewProjects/UnnamedFileSystemAutomationProject/docs/rfcs/RFC-0004_Runtime_Engine_Event_Bus_and_Debouncing.md)
