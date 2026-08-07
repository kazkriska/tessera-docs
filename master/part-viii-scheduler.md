# Part VIII — Scheduler

## 1. Purpose
Own all execution *policy*: queues, worker pools, priorities, retries, delays, rate limits, lock
acquisition, and dependency ordering. Sits between the Event Bus and the Dispatcher so scheduling
logic does not leak into either.

## 2. Motivation
The user considered simply "queue with configurable workers" and agreed a dedicated Scheduler stage
was needed (ChatGPT's clarifying Q3, accepted). Without it, the Dispatcher accumulates queueing,
retry, and lock logic — a known anti-pattern.

## 3. Responsibilities
- Maintain one **sequential per-workspace queue**; workspaces execute **concurrently**.
- Enforce **ticket-level and action-level locks** around state mutations (user Q19).
- Apply `retry:` and `timeout:` from the executable descriptor.
- Honor dependency relationships (`depends_on`) when ordering.
- Debounce rapid duplicate events.
- Drain gracefully on shutdown.

## 4. Design

### 4.1 Queue topology
- Each **Workspace** (the external workspace a Ticket points to) has its own queue.
- Within a workspace queue, hooks/actions execute **sequentially** (user Q7).
- Workspace queues run **concurrently** with one another.
- Fallback if topology is too complex: a single queue with configurable worker count, still respecting
  ticket/action locks (user's stated fallback).

### 4.2 Locking (Invariant I-8)
- **Ticket-level lock:** serializes state mutations for one Ticket (e.g. `locks/HQ_BR-001.lock`).
- **Action-level lock:** prevents a long action (e.g. `delegate`) from starting twice
  (`locks/HQ_BR-001.delegate.lock`).
- Locks are acquired only around state mutations, never around pure reads.
- Stale locks (from a crashed runtime) are reaped on boot.

### 4.3 Retry / timeout
From the manifest descriptor: `timeout:` caps a single run; `retry:` sets max attempts with
backoff. Exhausted retries record a failure in `activity.jsonl` and emit `ticket.action.failed`.

### 4.4 Dependency ordering
If Ticket A `depends_on` B, A's `Ready`/execution waits until B reaches a compatible state
(`Completed` or as configured). The Scheduler consults the relationship index (Part IV).

### 4.5 Debounce
Rapid `metadata.updated` bursts are coalesced within a configurable window so a hook runs once, not N
times.

## 5. Directory Layout
Queue state is in-memory; durable hints may live in `.ticket-runtime/cache/`. Lock files under
`.ticket-runtime/locks/`.

## 6. Lifecycle
Scheduler starts with the runtime; queues populate as events arrive; drains on shutdown.

## 7. Interaction
- **In:** Event Bus deliveries (enqueue requests).
- **Out:** Dispatcher (hand off a ready executable).
- **With:** State Manager (locks, transitions), Registry (dependencies), Permissions (pre-check).

## 8. Failure Modes
| Failure | Handling |
| --- | --- |
| Lock contention | Requester waits/requeues; deadlock avoided by acyclic dependency rule. |
| Worker crash | Task marked failed; retried per policy; queue continues. |
| Shutdown mid-run | Graceful drain; in-progress tasks complete or roll back under lock. |

## 9. Security
Pre-execution permission check occurs here (or just before Dispatch). Locks prevent concurrent
mutation that could violate integrity. Escalation prompts (Part IX) may block a task until approved.

## 10. Future Extensions
- **Distributed scheduler:** workers on multiple nodes share the registry (deferred).
- **Priority bands:** `priority:` field on actions influences ordering within a queue.

## 11. Examples
Ticket A points to workspace `/w/a`; Ticket B to `/w/b`. Both `metadata.updated` fire. A's queue and
B's queue run concurrently. Within A's queue, two rapid edits debounce to one `update_summary.py`
run, serialized behind A's ticket lock.

## 12. Rationale
Per-workspace sequential queues match the user's exact Q7 answer and keep external work directories
from interleaving unsafely, while concurrency across workspaces preserves throughput. Putting this in
its own stage keeps the Dispatcher a pure "choose runner" function and the Bus a pure fan-out — each
component has one job.

---

> **REVISION — Appended from FRAME documentation (Rev A · 2026-08-07).**
> *Non-destructive: all prior text in this document is unchanged and remains canonical. This block augments it with material drawn from the FRAME spec set (same design, independent authorship). Status: Appended.*

### R.A.8 — Priority bands & concurrency caps (from FRAME Ch.1 / Ch.4)

FRAME assigns explicit priority levels (Tessera's §4.1 per-workspace queue can adopt these as an ordering key):

1. `0` Emergency / cancellation handlers
2. `1` Direct CLI / user action commands
3. `2` File-modification hooks
4. `3` Background asset indexing / maintenance

The worker pool caps concurrent subprocesses to prevent CPU saturation during cascading events. Priority sorts by `(priority_level, timestamp)`; the recursion guard (Part VII R.A.7) aborts deep chains.
