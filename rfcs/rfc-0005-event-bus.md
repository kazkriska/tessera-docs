# RFC-0005 — Event Bus

- **Status:** Ratified
- **Version:** 1.0
- **References:** Master Part VII; RFC-0000 (I-4, I-5); RFC-0001; RFC-0006

## Summary
In-process publish/subscribe channel carrying **domain events**. Decouples producers from consumers and
enforces notification-only semantics.

## Event taxonomy
- **Filesystem (translated):** `metadata.updated`, `asset.added`, `env.changed`
- **Lifecycle:** `ticket.created`, `ticket.ready`, `ticket.completed`, `ticket.archived`, …
- **Relationship:** `parent.changed`, `dependency.satisfied`
- **Custom/user:** any name declared by a Ticket (via action `emit`)
- **Runtime:** `ticket.discovered`, `ticket.invalid`, `runtime.started`

## Envelope
```json
{ "id":"uuid", "name":"metadata.updated", "ticket_id":"HQ_BR-001",
  "ts":"2026-08-07T10:00:00Z", "data": {"path":"metadata.json","change":"modified"} }
```

## Subscription model
- Hooks subscribe by event `name` (from manifest `hooks:`).
- A Ticket subscribes to events whose `ticket_id` matches, plus relationship-scoped events.
- Delivery is asynchronous via the Scheduler; never blocks the publisher.

## Notification-only (I-4)
The bus does **not** return values. A hook needing a result invokes an **action**; that action may
emit a follow-up event. This keeps data flow acyclic and debuggable.

## Failure modes
Subscriber throws → isolated, bus continues, routed to retry+log. Duplicate events → debounced by
Scheduler. Unknown name → delivered to wildcard/relationship subscribers or ignored.

## Rationale
Realizes "everything is event-driven" and "extensibility over hardcoding": new behavior attaches by
subscribing, not by editing central code.
