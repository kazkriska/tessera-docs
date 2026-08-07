# RFC-0006: State Machine and Graph Dependency Resolution

- **RFC Number:** RFC-0006
- **Title:** Ticket Lifecycle State Machine and Dependency Graph Unblocking
- **Status:** Draft / Active
- **Author:** FRAME Core Architecture Team
- **Created:** 2026-08-07
- **Replaces:** None
- **Category:** Standards Track / Lifecycle

---

## 1. Abstract

This RFC formalizes the 9 lifecycle statuses of a FRAME Ticket, rules for valid state transitions, agent/human handoff semantics, and parent-child dependency graph auto-unblocking.

---

## 2. State Machine Specification

```
[* ] -> created -> initializing -> ready -> running -> completed -> archived -> [*]
                                              |----> blocked ----|
                                              |----> handoff ----|
                                              |----> failed -----|
```

The 9 states are:
1. `created`: Directory bundle initialized on disk.
2. `initializing`: Workspace setup scripts executing.
3. `ready`: Validated, awaiting triggers.
4. `running`: Action or hook handler currently executing.
5. `blocked`: Waiting on child tickets or external dependencies.
6. `handoff`: Control temporarily assigned to human or AI agent.
7. `completed`: Work items finished, assets generated.
8. `failed`: Execution error or timeout occurred.
9. `archived`: Read-only terminal state.

---

## 3. Parent-Child Graph Unblocking

When a Parent Ticket delegates tasks to Child Tickets:
1. Parent transitions to `status: blocked` with `relationships.dependencies = ["child-1", "child-2"]`.
2. When all child tickets transition to `status: completed`, the runtime automatically emits `dependency.resolved` to the parent.
3. Parent transitions back to `status: ready` to resume execution pipeline.

---

## 4. References & Related RFCs

- [RFC-0002: Ticket Resource Bundle Specification](file:///home/kaz/NewProjects/UnnamedFileSystemAutomationProject/docs/rfcs/RFC-0002_Ticket_Resource_Bundle_Specification.md)
