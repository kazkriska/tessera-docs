# RFC-0000: FRAME Philosophy and Guiding Principles

- **RFC Number:** RFC-0000
- **Title:** FRAME Philosophy, Vision, and Invariants
- **Status:** Draft / Active
- **Author:** FRAME Core Architecture Team
- **Created:** 2026-08-07
- **Replaces:** None
- **Category:** Standards Track / Core

---

## 1. Abstract

This RFC defines the foundational philosophy, architecture vision, non-goals, and core invariants of **FRAME** (*Filesystem Resource Automation & Management Engine*). FRAME is a filesystem-native, resource-agnostic automation framework that treats directories as self-contained, active objects ("Tickets") possessing data, metadata, state, scripts, reactive triggers, and declarative contracts.

---

## 2. Motivation

Traditional task, workflow, and AI agent frameworks rely on centralized databases (PostgreSQL, Redis) or SaaS APIs, coupling execution to complex external infrastructure. Conversely, static filesystems store rich data and assets but lack reactivity—folders cannot observe changes, maintain lifecycle state, or trigger autonomous action handlers.

FRAME unifies data, metadata, state, scripts, and triggers into portable `.ticket` directory bundles monitored by a decoupled runtime controller (`framed`).

---

## 3. Core Terminology

- **FRAME**: Filesystem Resource Automation & Management Engine.
- **Ticket (`*.ticket`)**: The fundamental resource unit in FRAME—a directory bundle containing state, metadata, logs, payload assets, and a declarative manifest.
- **Manifest (`MANIFEST.yaml`)**: The declarative contract defining identity, file watches, event hooks, and callable actions.
- **Runtime Daemon (`framed`)**: The lightweight background process monitoring workspaces and dispatching event handlers.
- **Ephemeral Projection**: The in-memory or SQLite index used by the runtime for fast lookups. It is 100% rebuildable from disk.

---

## 4. System Invariants

All components in the FRAME ecosystem MUST adhere to these five non-negotiable invariants:

1. **Filesystem Sovereignty**: The filesystem is the single primary source of truth. Databases or indexes are purely ephemeral read-side caches.
2. **Declarative / Imperative Boundary**: `MANIFEST.yaml` declares WHAT events a Ticket monitors and WHAT capabilities it exposes; `scripts/` implement HOW execution occurs.
3. **Cold-Boot Rebuildability**: Deleting the runtime cache or index MUST NOT cause data loss. Re-scanning disk manifests fully restores the runtime graph state.
4. **Subprocess Isolation**: Script handlers execute as isolated subprocesses with explicit timeouts, environment variable masking, and process group boundaries.
5. **Debouncing & Idempotency**: Filesystem events are debounced ($300\text{ms}$), and event contexts include trace IDs to ensure safe, idempotent execution.

---

## 5. Security & Sandbox Strategy

Execution of hooks and actions uses **direct POSIX subprocess invocation with environment masking and path jailing**, providing a high-performance, lightweight runtime footprint without mandating heavy container runtimes (e.g. Docker).

---

## 6. References & Related RFCs

- [RFC-0001: Filesystem-Native Layered Architecture](file:///home/kaz/NewProjects/UnnamedFileSystemAutomationProject/docs/rfcs/RFC-0001_Filesystem_Native_Layered_Architecture.md)
- [RFC-0002: Ticket Resource Bundle Specification](file:///home/kaz/NewProjects/UnnamedFileSystemAutomationProject/docs/rfcs/RFC-0002_Ticket_Resource_Bundle_Specification.md)
