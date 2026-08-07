# RFC-0007: CLI and Python SDK Interface

- **RFC Number:** RFC-0007
- **Title:** `frame` Command-Line Interface and `frame-sdk` Library Contract
- **Status:** Draft / Active
- **Author:** FRAME Core Architecture Team
- **Created:** 2026-08-07
- **Replaces:** None
- **Category:** Standards Track / Interfaces

---

## 1. Abstract

This RFC specifies the public CLI commands (`frame`) and Python SDK interfaces (`frame-sdk`) for programmatic and interactive operational control of Ticket resources.

---

## 2. CLI Command Suite (`frame`)

- `frame init [PATH]`: Initialize workspace structure.
- `frame watch [PATH]`: Launch daemon to monitor workspace.
- `frame list [--kind KIND] [--json]`: List discovered tickets.
- `frame create <ID> --kind KIND`: Scaffold new ticket bundle.
- `frame inspect <ID>`: Print ticket metadata, state, and recent activity.
- `frame run <ID> <ACTION> [--input KEY=VAL]`: Trigger named action.
- `frame reset <ID>`: Recover ticket from failed state.

---

## 3. Python SDK Contract (`frame-sdk`)

```python
from frame_sdk import Ticket

ticket = Ticket.from_path("./HQ_BR-001.ticket")

# Context manager ensures POSIX lock and atomic write
with ticket.mutate_state() as state:
    state.status = "running"
    state.step = "data_processing"

# Append to activity.jsonl with flock
ticket.log_activity("step.completed", details={"step": "data_processing"})
```

---

## 4. References & Related RFCs

- [RFC-0002: Ticket Resource Bundle Specification](file:///home/kaz/NewProjects/UnnamedFileSystemAutomationProject/docs/rfcs/RFC-0002_Ticket_Resource_Bundle_Specification.md)
