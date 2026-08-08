# Part XI — SDK Architecture

## 1. Purpose
Describe the software development kit that lets programs (and the CLI) drive Tessera programmatically:
discover tickets, read state, request transitions, and invoke actions. Because the CLI is built on the
runtime's own functions, shipping the SDK is nearly free (user Q23).

## 2. Motivation
External tools, agents, and future subsystem runtimes need a stable programmatic interface rather than
shelling out to the CLI. The SDK is the contract for "control Tessera from code."

## 3. Responsibilities
- Connect to a running runtime (over `runtime.sock`) or operate in direct/filesystem mode.
- Expose typed operations: `discover`, `get_ticket`, `transition`, `invoke_action`, `emit`, `subscribe`.
- Hide transport details (socket framing, serialization) behind a clean API.
- Reuse the exact same runtime functions the CLI uses.

## 4. Design

### 4.1 Layered architecture
```
Application / Agent
        │
        ▼
   Tessera SDK  (Python client)
        │  (JSON-RPC-style over runtime.sock, or direct mode)
        ▼
   Runtime  (lib/ticket-management)
        │
        ▼
   Tickets (filesystem source of truth)
```

### 4.2 Client API (sketch)
```python
from tessera import Runtime

rt = Runtime.connect(sock="/FrameworkRoot/Tickets/.ticket-runtime/runtime.sock")
# or direct mode:
# rt = Runtime.direct(repo="/FrameworkRoot/Tickets")

tickets = rt.discover()                 # list registered tickets
t = rt.get_ticket("HQ_BR-001")
t.transition("Running")                 # validated per Part VI
t.invoke_action("delegate", assignee="bob")   # emits event, may prompt
for ev in rt.subscribe(["metadata.updated"]):
    print(ev.ticket_id, ev.name)
```

### 4.3 Transport
- **Attached mode:** framed JSON over Unix domain socket. Preferred; enforces singleton and lets the
  SDK drive the live runtime.
- **Direct mode:** SDK calls runtime functions against the filesystem without a running daemon
  (read-only discovery, manifest validation, offline edits). Mutations requiring the bus/scheduler
  need a running runtime.

### 4.4 Reuse
The CLI (`cli.py`) imports the same `Runtime` client; there is no duplicated logic. This satisfies the
user's expectation that the SDK ships easily because the CLI already wraps the functions.

## 5. Directory Layout
SDK package (`tessera/` ) published alongside `lib/ticket-management/`. `pyproject.toml` declares it.

## 6. Lifecycle
SDK objects are short-lived clients; the runtime they talk to has its own lifecycle (Part II §6).

## 7. Interaction
- **With runtime:** socket or direct.
- **With agents (future):** an agent is just another SDK client; delegation/handoff is an action that
  may itself use the SDK (conceptualized now, full contract deferred per Charter Non-Goals).
- **With CLI:** shared client.

## 8. Failure Modes
| Failure | Handling |
| --- | --- |
| Socket missing | SDK raises `RuntimeNotRunning`; suggest `runtime start` or use direct mode. |
| Invalid transition/action | SDK returns typed error mirroring runtime validation. |
| Permission required | SDK surfaces an approval callback hook the caller implements. |

## 9. Security
The SDK enforces the same permission gates as the CLI (Part IX). An SDK caller cannot exceed what the
runtime grants; escalation still requires approval.

## 10. Future Extensions
- **Language bindings:** the socket protocol is language-agnostic, so a Go/Rust/JS SDK can be added.
- **Async client:** `async` API for event streaming.
- **Agent SDK:** a higher-level client for delegation/orchestration (when the agent contract is specified).

## 11. Examples
See §4.2. An agent uses `rt.invoke_action("delegate", …)` to hand a ticket to a sub-agent, then
subscribes to `ticket.completed` to learn when it finishes — entirely event-driven, no polling.

## 12. Rationale
Defining the SDK alongside the CLI (not later) turns the "functions the CLI uses" into a first-class,
stable contract. A socket-based, language-agnostic transport means future runtimes and external tools
integrate without forking the runtime. This directly serves the user's Q23 ("ship the complementing
SDK easily") and the broader multi-runtime vision.

---

> **REVISION — Layout rename (Rev B · 2026-08-08).**
> *Non-destructive: prior text unchanged, remains canonical. The package tree moved to a src/ layout: `lib/ticket-management/` → `src/tessera_runtime/`; `tessera/` (SDK) → `src/tessera_sdk/`. Imports: `from tessera import …` → `from tessera_sdk import …`; `ticket_management.cli:main` → `tessera_sdk.cli:main`. Status: Appended.*

### R.B.1 — Affected references in this document

The rows below map canonical text above (left) to its Rev B equivalent (right). The canonical text is **not** rewritten; read it through this table.

| Location (canonical text) | As written (Rev A, canonical) | Rev B equivalent |
|---|---|---|
| § 4 — Layering diagram | `Runtime  (lib/ticket-management)` | `Runtime  (src/tessera_runtime)` |
| § 5 — Usage example | `from tessera import Runtime` | `from tessera_sdk import Runtime` |
| § 8 — Packaging | SDK package (`tessera/`) published alongside `lib/ticket-management/` | `src/tessera_sdk/` published alongside `src/tessera_runtime/` |
| § 8 — Packaging | `ticket_management.cli:main` | `tessera_sdk.cli:main` |

No canonical sentence above is amended by this block; tooling that resolves paths or imports MUST apply the mapping table. Rev A text remains the authority on *behaviour*; Rev B is the authority on *location*.
