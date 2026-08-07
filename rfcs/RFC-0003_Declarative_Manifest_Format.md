# RFC-0003: Declarative Manifest Format (`MANIFEST.yaml`)

- **RFC Number:** RFC-0003
- **Title:** MANIFEST.yaml Declarative Schema v1
- **Status:** Draft / Active
- **Author:** FRAME Core Architecture Team
- **Created:** 2026-08-07
- **Replaces:** None
- **Category:** Standards Track / Schema

---

## 1. Abstract

This RFC specifies the normative syntax and validation rules for `MANIFEST.yaml` files inside FRAME Tickets. It defines the Kubernetes-inspired schema format (`apiVersion: ticket/v1`, `kind: Ticket`), file watching rules, event hooks, and named action interfaces.

---

## 2. Manifest Schema (`apiVersion: ticket/v1`)

```yaml
apiVersion: ticket/v1
kind: Ticket

metadata:
  id: TICKET_UNIQUE_ID
  name: "Human Readable Name"
  version: "1.0.0"
  tags: [tag1, tag2]

spec:
  runtime:
    python: ">=3.12"
    toolchain: uv

  env:
    KEY: VALUE

  watch:
    - path: "metadata.json"
      events: [modify]
      trigger: metadata_updated

  hooks:
    ticket.initialized:
      - entrypoint: scripts/setup_workspace.py
        timeout: 60
        async: false

  actions:
    action-name:
      description: "Action description"
      entrypoint: scripts/handler.py
      timeout: 30
      retry: 2
      permissions: [read:ticket, write:state]
      inputs:
        param1: { type: string, required: true }
      outputs:
        result: { type: string }

  exports:
    - task.status
```

---

## 3. Uniform Handler Specification

Both `hooks` and `actions` map execution targets to a uniform handler object containing:
- `entrypoint` (required): Relative path to execution script.
- `timeout` (optional, default 30): Max execution time in seconds.
- `retry` (optional, default 0): Number of automatic retries on failure.
- `async` (optional, default true): Whether runner executes asynchronously in background worker pool.
- `permissions` (optional, default []): Array of granted capability scopes.

---

## 4. References & Related RFCs

- [RFC-0005: Subprocess Execution and Sandboxing](file:///home/kaz/NewProjects/UnnamedFileSystemAutomationProject/docs/rfcs/RFC-0005_Subprocess_Execution_and_Sandboxing.md)
