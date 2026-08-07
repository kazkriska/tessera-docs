# FRAME Master Architecture Specification
## Chapter 7: CLI & SDK Specification

---

## 1. Overview

The FRAME framework exposes a command-line interface (**`frame` CLI**) and a Python library (**`frame-sdk`**) for human engineers, scripts, and AI agents to interact with Ticket resources.

---

## 2. CLI Specification (`frame`)

Built using Python `Typer` and `Rich` for clear output.

### 2.1 Workspace & Daemon Commands

```bash
# Initialize a new FRAME workspace in the current directory
frame init [PATH]

# Start the FRAME runtime daemon to watch designated workspace(s)
frame watch [PATH] [--daemon] [--log-level INFO]

# List all discovered tickets in the workspace
frame list [--kind KIND] [--status STATUS] [--json]
```

### 2.2 Ticket Lifecycle & Action Commands

```bash
# Create a new self-contained .ticket bundle from a template
frame create <TICKET_ID> --kind ticket --title "Report Generation"

# Inspect metadata, state, manifest, and activity log of a ticket
frame inspect <TICKET_ID>

# Trigger a specific named action on a ticket
frame run <TICKET_ID> <ACTION_NAME> [--input KEY=VAL]

# Reset or recover a failed ticket
frame reset <TICKET_ID>
```

---

## 3. Python SDK Specification (`frame-sdk`)

The `frame-sdk` provides programmatic interaction with Tickets, ensuring atomic writes and audit logging.

### 3.1 Code Example

```python
from frame_sdk import Ticket, Workspace

# 1. Load Ticket resource by path
ticket = Ticket.from_path("./HQ_BR-001.ticket")

# 2. Inspect state
print(f"Status: {ticket.state.status}, Step: {ticket.state.step}")

# 3. Perform atomic state update
with ticket.mutate_state() as state:
    state.status = "running"
    state.step = "generating_infographic"
    state.variables["progress"] = 50

# 4. Append audit log entry
ticket.log_activity(
    event="infographic.started",
    details={"asset": "task/assets/infographic.png"}
)

# 5. Invoke declared action
result = ticket.run_action("configure-assignee", inputs={"assignee_id": "agent_alpha", "assignee_type": "agent"})
```

---

## 4. Agent IPC & Integration Protocol

When an AI Agent (e.g. Antigravity or a custom subagent) operates on a Ticket:
1. **Context Loading**: The agent reads `MANIFEST.yaml`, `metadata.json`, and `state.json`.
2. **Execution**: The agent invokes `frame run <TICKET_ID> <ACTION_NAME>` via standard terminal command or imports `frame_sdk`.
3. **Log Ingestion**: All outputs produced by the agent are appended to `activity.jsonl` with actor payload `{"id": "agent_name", "type": "agent"}`.

---

## 5. Rationale Matrix

| Component | WHAT | WHY | HOW |
| :--- | :--- | :--- | :--- |
| `frame-sdk` Context Manager | Use `with ticket.mutate_state()` syntax for state updates. | Enforces atomic writes (`os.replace`) and automatic file locking (`flock`) behind an intuitive Python interface. | Automatically acquires `.lock`, yields mutable state copy, writes to temp file, and swaps upon context exit. |
| Typer + Rich CLI | Build CLI using Python Typer and Rich. | Provides rich interactive formatting for humans while supporting `--json` flags for agent automation. | Output automatically detects terminal TTY or formats structured JSON output when piped. |
