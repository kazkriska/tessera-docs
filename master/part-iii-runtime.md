# Part III — Runtime Internals

## 1. Purpose
Describe the runtime daemon: its modules, boot/shutdown sequence, and how the pipeline stages
(Discovery → Watcher → Event Bus → Scheduler → Dispatcher → Executor) are wired together. This is the
"how the engine is built" Part; Part II is the "what the engine is for" map.

## 2. Motivation
A single long-running Python 3.12+ daemon (via `uv`) owns one TicketRepository. Concretely specifying
its modules prevents the "one giant watcher script" anti-pattern and keeps each stage replaceable.

## 3. Responsibilities
- Boot: load config, enforce singleton, open/repair registry.
- Discover Tickets and build the registry.
- Watch the repository filesystem.
- Operate the Event Bus, Scheduler, Dispatcher, and Executors.
- Manage state, activity logs, locks, and environment resolution.
- Serve CLI/SDK over the runtime socket.

## 4. Design

### 4.1 Module layout (`lib/ticket-management/`)
```
runtime/
  watcher.py     # inotify over repository -> filesystem events
  dispatcher.py  # chooses runner/executor for a hook/action
  manifest.py    # load + validate MANIFEST.yaml
  executor.py    # runs a declared executable via a runner
  state.py       # lifecycle transitions + state.json persistence
  env.py         # .env resolution (global -> ticket) + injection
plugins/
  python_runner.py
  bash_runner.py
  node_runner.py
cli.py           # Typer-based CLI; talks to runtime over socket
pyproject.toml   # uv-managed; Python >=3.12
```

### 4.2 Boot sequence
1. Load `Tickets/.ticket-runtime/config.yaml`.
2. Singleton check: if `runtime.sock` exists and responds, attach (CLI/SDK mode) and do not watch.
   Otherwise create socket + (fallback) `runtime.pid`.
3. Open `registry.db`; if missing/corrupt, recreate (derived).
4. **Discovery:** scan `*.ticket/` directories; load + validate manifests; register Tickets; build
   relationship index.
5. Start **Watcher** (inotify) over the repository.
6. Enter **Run** loop: process events → bus → scheduler → dispatcher → executor. Serve socket.
7. On SIGTERM/SIGINT: drain scheduler queues (graceful), release locks, remove socket/pid.

### 4.3 Wiring
- **Watcher** → emits `fs:<path>:<event>` to an internal adapter that translates to domain events and
  publishes on the **Event Bus**.
- **Event Bus** → subscribers are Scheduler enqueue requests keyed by `(ticket_id, event_name)`.
- **Scheduler** → orders/enforces policy; on ready, calls **Dispatcher** with the executable descriptor.
- **Dispatcher** → reads `shell`, selects the matching **runner** plugin, calls **Executor**.
- **Executor** → invokes the runner with resolved env, captures exit code/stdout/stderr, reports result.

### 4.4 Environment resolution (`env.py`)
Resolution order (later overrides earlier): Global `Tickets/.env` → Ticket `HQ_BR-001.ticket/.env`.
Manifest `env.inherit: false` suppresses the global layer. The resulting flat env is passed to the
runner. Secrets stay in `.env` (gitignored in practice); they are never written to `activity.jsonl`.

## 5. Directory Layout
See §4.1 and the Master index.

## 6. Lifecycle
The runtime process lifecycle is in Part II §6. Ticket lifecycle is Part VI.

## 7. Interaction
- With Tickets: read manifests/metadata; write only `state.json`, `activity.jsonl`, and hook/action
  outputs under granted permissions.
- With CLI/SDK: over `runtime.sock` (JSON-RPC-style or simple framed protocol).
- With itself: all stages communicate via in-process channels; the bus is in-process.

## 8. Failure Modes
| Failure | Handling |
| --- | --- |
| `registry.db` corrupt | Recreate + rescan (derived). |
| Watcher drops events (rapid edits) | Scheduler debounce + periodic reconcile rescan. |
| Runner crash | Executor captures non-zero exit; Scheduler retries per `retry:`; records failure. |
| Socket stale (crashed runtime) | Boot reaps stale socket/pid, starts fresh. |
| Unhandled exception in hook | Isolated per-execution; runtime continues; failure logged + `activity.jsonl` entry. |

## 9. Security
All privileged mutation flows through the Scheduler + Permission system (Part IX). The runtime never
runs arbitrary code except declared executables under a runner with a scoped environment. Lock files
prevent concurrent mutation (Part IX/VIII).

## 10. Future Extensions
- **Rust/Go runtime:** replace `lib/ticket-management/` internals; Ticket format & manifest unchanged.
- **Distributed workers:** Scheduler delegates to remote executors (deferred).
- **Multiple repositories:** one runtime may later watch several repositories with separate registries.

## 11. Examples
See Part II §11 for the end-to-end `metadata.updated` flow through these modules.

## 12. Rationale
Splitting into focused modules (rather than one script) honors "extensibility over hardcoding" and
lets each stage evolve. The boot order makes the registry rebuildable and the singleton enforceable,
which together satisfy Invariants I-2, I-6, I-9. `uv` provides reproducible Python without making
`venv` responsible for behavior (the original misconception).
