# RFC-0012 — Implementation Roadmap

- **Status:** Ratified
- **Version:** 1.0
- **References:** Master Part XII; RFC-0000 … RFC-0011

## Summary
Phased, priority-ordered plan to build Tessera from the spec. Each phase yields a runnable, testable
system and de-risks the hardest parts early.

## Phases
- **A — Foundation:** scaffold `FrameworkRoot/`; `manifest.py` loader/validator; json/state/activity
  models. *Done:* manifest validates, Ticket parseable + registry-insertable.
- **B — Discovery & Registry:** recursive `*.ticket/` scan; `registry.db` (derived); relationship
  index. *Done:* rescan rebuilds equivalent registry; dangling refs handled.
- **C — Watcher & Event Bus:** inotify watcher; adapter → domain events; in-process bus fan-out.
  *Done:* editing metadata.json emits `metadata.updated` to subscribers.
- **D — Scheduler, Dispatcher, Executor:** per-workspace sequential queues; concurrent workspaces;
  ticket/action locks; runner selection + python/bash/node runners; retry/timeout/debounce. *Done:*
  hook runs under lock with retries; rapid edits debounce.
- **E — Lifecycle & State:** `state.py` transitions enforcing RFC-0007; emit lifecycle events;
  persist state.json + activity.jsonl. *Done:* full path enforced + logged.
- **F — Permissions:** capability model + safe defaults; escalation prompts; executor enforcement.
  *Done:* `network:true` pauses for approval; denied capabilities blocked.
- **G — CLI & SDK:** Typer CLI over socket; Python SDK (shared `Runtime`). *Done:* `runtime start`,
  `create`, `action`, `inspect` work end-to-end.
- **H — Hardening & Docs:** crash recovery (lock reaping, registry repair), logging, tests, RFC
  extraction. *Done:* `kill -9` → restart recovers; tests green.

## Deferred (Charter Non-Goals)
Agent orchestration contract; AI provider integrations; general plugin marketplace; compiled Ticket
packages; Windows/macOS; cross-machine distributed scheduling; additional repositories.

## Rationale
Order front-loads the riskiest, architecture-defining components (watch → bus → schedule → execute →
lifecycle → permissions) so the system is demonstrable early and CLI/SDK are thin wrappers. Explicit
deferrals keep v1 scope honest. This roadmap bridges the Master to the RFC suite: each RFC versions
independently from here.
