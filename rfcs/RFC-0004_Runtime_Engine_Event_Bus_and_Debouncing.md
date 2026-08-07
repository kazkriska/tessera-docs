# RFC-0004: Runtime Engine Event Bus and Debouncing

- **RFC Number:** RFC-0004
- **Title:** Event Engine, Debouncing Mechanics, and Priority Scheduler
- **Status:** Draft / Active
- **Author:** FRAME Core Architecture Team
- **Created:** 2026-08-07
- **Replaces:** None
- **Category:** Standards Track / Runtime

---

## 1. Abstract

This RFC standardizes the event translation pipeline, sliding-window debouncing algorithms, and priority task scheduling within the FRAME runtime daemon (`framed`).

---

## 2. Event Debouncing Algorithm

Filesystem monitoring tools emit noisy streams of file modification events during file saves. The FRAME Event Engine aggregates raw OS events using a **$300\text{ms}$ Sliding-Window Debouncer**:

```python
# Debouncing logic pseudo-code
class SlidingWindowDebouncer:
    def __init__(self, window_ms: int = 300):
        self.window_sec = window_ms / 1000.0
        self.pending_events = {}

    def on_fs_event(self, filepath: str, event_type: str):
        # Reset or extend timer window for filepath
        self.pending_events[filepath] = time.time() + self.window_sec

    def flush_expired(self) -> list[str]:
        now = time.time()
        ready = [fp for fp, exp in self.pending_events.items() if now >= exp]
        for fp in ready:
            del self.pending_events[fp]
        return ready
```

---

## 3. Priority Task Queue

Triggered event handlers are placed into a thread-safe `PriorityQueue` with 4 operational bands:
- `Band 0 (Critical)`: System emergency recovery & process cancellation handlers.
- `Band 1 (Interactive)`: Direct user or agent CLI invocation (`frame run`).
- `Band 2 (Hook Handlers)`: File modification event hooks.
- `Band 3 (Maintenance)`: Background asset indexing and cache cleanup.

---

## 4. References & Related RFCs

- [RFC-0001: Filesystem-Native Layered Architecture](file:///home/kaz/NewProjects/UnnamedFileSystemAutomationProject/docs/rfcs/RFC-0001_Filesystem_Native_Layered_Architecture.md)
