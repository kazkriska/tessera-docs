# RFC-0010 — SDK Architecture

- **Status:** Ratified
- **Version:** 1.0
- **References:** Master Part XI; RFC-0004; RFC-0009; RFC-0008

## Summary
Python SDK that lets programs/agents drive Tessera programmatically. Because the CLI wraps the
runtime's own functions, the SDK ships nearly for free (user Q23).

## Layered architecture
```
Application / Agent
   → Tessera SDK (Python client)
   → Runtime (socket or direct mode)
   → Tickets (filesystem)
```

## Client API (sketch)
```python
from tessera import Runtime
rt = Runtime.connect(sock="/FrameworkRoot/Tickets/.ticket-runtime/runtime.sock")
# or rt = Runtime.direct(repo="/FrameworkRoot/Tickets")
tickets = rt.discover()
t = rt.get_ticket("HQ_BR-001")
t.transition("Running")                 # validated (RFC-0007)
t.invoke_action("delegate", assignee="bob")   # emits event, may prompt
for ev in rt.subscribe(["metadata.updated"]):
    print(ev.ticket_id, ev.name)
```

## Transport
- **Attached:** framed JSON over Unix socket (preferred; enforces singleton).
- **Direct:** SDK calls runtime functions on the filesystem without a daemon (read-only discovery,
  validation, offline edits). Mutations needing bus/scheduler require a running runtime.

## Reuse
CLI imports the same `Runtime` client — no duplicated logic.

## Failure modes
Socket missing → `RuntimeNotRunning` (suggest start or direct mode). Invalid op → typed error.
Permission required → SDK exposes an approval callback the caller implements.

## Future
Language bindings (socket is language-agnostic); async client; higher-level Agent SDK (when the agent
contract is specified — deferred per Charter Non-Goals).

## Rationale
Defining the SDK alongside the CLI turns "functions the CLI uses" into a stable contract. A socket-
based, language-agnostic transport lets future runtimes and external tools integrate without forking
the runtime — serving the multi-runtime vision.

---

> **REVISION — Layout rename (Rev B · 2026-08-08).**
> *Non-destructive: prior text unchanged, remains canonical. The package tree moved to a src/ layout: `lib/ticket-management/` → `src/tessera_runtime/`; `tessera/` (SDK) → `src/tessera_sdk/`. Imports: `from tessera import …` → `from tessera_sdk import …`; `ticket_management.cli:main` → `tessera_sdk.cli:main`. Status: Appended.*

### R.B.1 — Affected references in this document

The rows below map canonical text above (left) to its Rev B equivalent (right). The canonical text is **not** rewritten; read it through this table.

| Location (canonical text) | As written (Rev A, canonical) | Rev B equivalent |
|---|---|---|
| § Usage example | `from tessera import Runtime` | `from tessera_sdk import Runtime` |
| § Packaging | SDK package `tessera/` | `src/tessera_sdk/` |

No canonical sentence above is amended by this block; tooling that resolves paths or imports MUST apply the mapping table. Rev A text remains the authority on *behaviour*; Rev B is the authority on *location*.
