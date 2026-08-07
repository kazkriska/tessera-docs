# Part X — Command-Line Interface

## 1. Purpose
Define the official CLI surface for operating Tessera: managing the runtime, repositories, tickets,
and invoking actions. The CLI is the primary human entry point and a thin client to the runtime.

## 2. Motivation
The user requested a basic CLI be defined (extensibility left for later, Q22). Because the CLI is
built on the runtime's own functions, an SDK follows almost for free (Part XI, Q23).

## 3. Responsibilities
- Start/stop/inspect the runtime (singleton via socket).
- Initialize and scan repositories.
- Create, inspect, and mutate tickets (lifecycle transitions, metadata edits).
- Invoke actions by name.
- Surface permission approval prompts.
- Render ticket state and activity.

## 4. Design
Built with **Typer** (`cli.py`). Commands delegate to runtime functions over `runtime.sock`; if no
runtime is running, some commands (e.g. `create`, `validate`) operate directly on the filesystem.

### 4.1 Command surface (v1)
```
ticket runtime start [--repo PATH]     # boot daemon (or attach if running)
ticket runtime stop                    # graceful shutdown via socket
ticket runtime status                  # is it running? queue depths? tickets?

ticket repo init [PATH]                # scaffold TicketRepository + .ticket-runtime/
ticket repo scan                       # force rediscovery + registry rebuild

ticket create <id> [--type TYPE]       # scaffold a new .ticket directory + minimal manifest
ticket inspect <id>                    # print metadata, state, manifest summary
ticket transition <id> <state>         # request a lifecycle transition (validated)
ticket action <id> <action> [args...]  # invoke a named action (emits event, may prompt)

ticket validate <id>                   # validate manifest against schema
ticket log <id> [--tail N]             # tail activity.jsonl
```

### 4.2 Invocation model
- `runtime start` creates/attaches to `runtime.sock`. A second `start` attaches and reports "already
  running" rather than spawning a competing watcher (Invariant I-6).
- `action`/`transition` send a request to the running runtime; the runtime performs validation,
  permission checks, and scheduling.

## 5. Directory Layout
`lib/ticket-management/cli.py` (Typer app). No dedicated on-disk state beyond what the runtime owns.

## 6. Lifecycle
The CLI process is transient; it talks to the long-lived runtime. `runtime start` begins the runtime
lifecycle (Part II §6).

## 7. Interaction
- **With runtime:** framed messages over `runtime.sock`.
- **With user:** stdout/stdio; approval prompts for permissions.
- **With SDK:** both wrap the same runtime functions (Part XI).

## 8. Failure Modes
| Failure | Handling |
| --- | --- |
| Runtime not running | Non-destructive commands run directly on FS; mutating ones error with "start runtime". |
| Invalid transition via CLI | Runtime rejects per Part VI rules; CLI reports reason. |
| Permission denied | CLI shows denial; records in activity.jsonl. |

## 9. Security
CLI runs as the invoking user; it triggers the runtime's permission gates (Part IX). It does not
bypass locks or capabilities.

## 10. Future Extensions
- `ticket plugin install <name>` (future plugin system).
- `ticket watch --since` event replay.
- Rich TUI dashboard.

## 11. Examples
```
$ ticket runtime start
Runtime started (pid 41285, sock /FrameworkRoot/Tickets/.ticket-runtime/runtime.sock)
$ ticket create HQ_BR-010 --type task
Created HQ_BR-010.ticket
$ ticket action HQ_BR-010 delegate --assignee bob
[permission] Allow subprocess? y
Action 'delegate' queued.
$ ticket inspect HQ_BR-010
id: HQ_BR-010  state: Delegated  assignee: bob
```

## 12. Rationale
A small, predictable CLI built directly on runtime functions keeps the human interface consistent with
the SDK and avoids a second implementation of behavior. Defining it in v1 (even if minimal) gives
operators a way to drive and debug the system immediately, which matters for a framework meant to be
implemented by others.
