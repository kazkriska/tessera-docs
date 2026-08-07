# RFC-0006 — Scheduler, Queues & Locking

- **Status:** Ratified
- **Version:** 1.0
- **References:** Master Part VIII; RFC-0000 (I-8); RFC-0001; RFC-0005; RFC-0007

## Summary
Owns all execution policy: queues, worker pools, priorities, retries, delays, rate limits, locks, and
dependency ordering. Sits between Event Bus and Dispatcher.

## Queue topology
- Each **Workspace** (external work dir a Ticket points to) has its own queue.
- Within a workspace queue, hooks/actions run **sequentially**.
- Workspace queues run **concurrently** with one another.
- Fallback: single queue with configurable workers, still respecting ticket/action locks.

## Locking (I-8)
- **Ticket-level:** `locks/HQ_BR-001.lock` serializes that Ticket's state mutations.
- **Action-level:** `locks/HQ_BR-001.delegate.lock` prevents a long action starting twice.
- Acquired only around state mutations, never pure reads.
- Stale locks (from crashed runtime) reaped on boot.

## Retry / timeout
From descriptor: `timeout:` caps one run; `retry:` sets max attempts with backoff. Exhausted →
`ticket.action.failed` in `activity.jsonl`.

## Dependency ordering
If A `depends_on` B, A's execution waits until B reaches a compatible state. Scheduler consults the
relationship index (RFC-0002).

## Debounce
Rapid duplicate events coalesced within a configurable window so a hook runs once.

## Failure modes
Lock contention → wait/requeue; acyclic dependency rule prevents deadlock. Worker crash → task failed,
retried, queue continues. Shutdown mid-run → graceful drain / rollback under lock.

## Rationale
Per-workspace sequential queues match the user's Q7; concurrency across workspaces preserves
throughput. A dedicated Scheduler keeps the Dispatcher a pure "choose runner" function.
