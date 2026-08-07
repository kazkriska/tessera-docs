# Part VII — Event Bus

## 1. Purpose
Define the in-process publish/subscribe channel that carries **domain events** between producers
(filesystem watcher, lifecycle transitions, actions) and consumers (hooks, scheduler). This is the
decoupling backbone of the framework.

## 2. Motivation
Without a bus, hooks call each other directly and the system becomes a tangle. The user explicitly
required: hooks do not call each other; they emit events (Invariant I-5). Events are notifications
only — anything needing a return value uses an action (Invariant I-4).

## 3. Responsibilities
- Accept publication of domain events.
- Deliver each event to all subscribed handlers (fan-out).
- Scope events to the repository and `lib/ticket-management/` (per user Q15).
- Remain fire-and-forget (notification semantics).

## 4. Design

### 4.1 Event taxonomy
| Class | Examples | Producer |
| --- | --- | --- |
| Filesystem (translated) | `metadata.updated`, `asset.added`, `env.changed` | Watcher adapter |
| Lifecycle | `ticket.created`, `ticket.ready`, `ticket.completed`, `ticket.archived` | State Manager |
| Relationship | `parent.changed`, `dependency.satisfied` | Registry/Scheduler |
| Custom/user | any `name` declared by a Ticket | Actions via `emit` |
| Runtime | `runtime.started`, `ticket.discovered`, `ticket.invalid` | Runtime |

### 4.2 Event envelope
```json
{
  "id": "uuid",
  "name": "metadata.updated",
  "ticket_id": "HQ_BR-001",
  "ts": "2026-08-07T10:00:00Z",
  "data": { "path": "metadata.json", "change": "modified" }
}
```
`data` is opaque to the bus; consumers interpret it.

### 4.3 Subscription model
- Hooks subscribe by event `name` (from manifest `hooks:`).
- A Ticket subscribes only to events whose `ticket_id` matches itself, plus optionally relationship-
  scoped events (e.g. a child reacts to `parent.changed`).
- Delivery is asynchronous via the Scheduler queue (never blocks the publisher).

### 4.4 Notification-only semantics
The bus does **not** return values to publishers. A hook that needs a result invokes an **action**
(via the runtime), which may itself emit a follow-up event. This keeps the data flow acyclic and
debuggable.

## 5. Directory Layout
In-process; no on-disk artifact. Event history is reflected in each Ticket's `activity.jsonl` when a
handler records it.

## 6. Lifecycle
The bus exists for the lifetime of the runtime process. On shutdown, in-flight events drain via the
Scheduler before exit.

## 7. Interaction
- **Producer:** Watcher adapter, State Manager, Actions (`emit`), Registry.
- **Consumer:** Scheduler (enqueues hook executions), runtime logging.
- **Never:** hooks calling hooks — only via emitted events.

## 8. Failure Modes
| Failure | Handling |
| --- | --- |
| Subscriber (hook) throws | Isolated; bus continues; failure routed to Scheduler retry + logged. |
| Duplicate events (watcher jitter) | Scheduler dedupes by (ticket_id, name, debounce window). |
| Unknown event name | Still delivered to any wildcard/relationship subscribers; otherwise ignored. |

## 9. Security
The bus is internal; only the runtime publishes. A Ticket cannot spoof another Ticket's events except
through its own granted actions. Permission checks happen at execution (Part IX), not at publish.

## 10. Future Extensions
- **Cross-repository bridging:** future runtimes publish to a shared bus (deferred).
- **Persistent event log:** optional durable replay stream for auditing.

## 11. Examples
Watcher sees `metadata.json` modified → adapter publishes `metadata.updated{ticket_id:HQ_BR-001}` →
bus fans out → Scheduler enqueues `scripts/update_summary.py` (declared under that hook).

## 12. Rationale
The bus realizes "everything is event-driven" (principle 6) and "extensibility over hardcoding"
(principle 7): new behavior attaches by subscribing, not by editing central code. Notification-only
semantics prevent hidden request/response coupling that would make the system untestable.
