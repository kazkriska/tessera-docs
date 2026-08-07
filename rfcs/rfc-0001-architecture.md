# RFC-0001 — Architecture & Component Pipeline

- **Status:** Ratified
- **Version:** 1.0
- **References:** Master Part II; RFC-0000; RFC-0004, RFC-0005, RFC-0006

## Summary
Defines the overall architecture: a single long-running Python 3.12+ runtime per TicketRepository,
with a linear, single-responsibility pipeline.

## Pipeline
```
Filesystem / External Event
   → Discovery → Registry (derived)
   → Watcher (inotify)
   → Event Bus (domain events)
   → Scheduler (policy)
   → Dispatcher (runner selection)
   → Executor (python/bash/node/…)
   → Ticket (state.json / activity.jsonl mutated under lock)
```

## Separation of concerns
- Discovery (startup/rescan) vs Watcher (ongoing).
- Watcher (filesystem events) vs Event Bus (domain events).
- Event Bus (fan-out) vs Scheduler (policy owner).
- Dispatcher (choose) vs Executor (run).

## Singleton
`runtime.sock` (preferred) or `runtime.pid` in `.ticket-runtime/` enforces one runtime per repository
(Invariant I-6). A second `ticket watch` attaches to the existing socket.

## Workspace vs Repository (terminology)
- **TicketRepository** = where Tickets live (`FrameworkRoot/Tickets/`). Canonical, editable.
- **Workspace** = external work directory referenced by a Ticket; runtime operates there only under
  granted permissions.

## Failure modes
Corrupt registry → rescan. Two runtimes → second attaches. Malformed manifest → flagged invalid.
Watcher miss → debounce + reconcile. Crash → restart rebuilds; stale locks reaped.

## Rationale
Single-responsibility stages let each be replaced (e.g. Rust runtime) without touching the Ticket
format or manifest contract. The explicit Scheduler prevents scheduling logic leaking into the
Dispatcher.
