# Part XII — Implementation Roadmap

## 1. Purpose
Provide a phased, priority-ordered plan for building Tessera from the specification. The roadmap turns
the design into actionable engineering milestones with clear done-criteria.

## 2. Motivation
A 100+ page spec is useless without a build order. The roadmap sequences work so each milestone yields
a runnable, testable system and de-risks the hardest parts (watching, scheduling, permissions) early.

## 3. Responsibilities
- Order milestones by dependency and risk.
- Define acceptance criteria per phase.
- Mark what is explicitly deferred (per Charter Non-Goals).
- Keep the Ticket runtime as the first shippable subsystem.

## 4. Design — Phases

### Phase A — Foundation (pre-runtime)
- Scaffold `FrameworkRoot/` (Tickets/, Skills/, lib/ticket-management/, README).
- `MANIFEST.yaml` schema + `manifest.py` loader/validator (reject anchors/aliases, enforce
  `apiVersion`/`kind`, id==basename).
- `metadata.json` / `state.json` / `activity.jsonl` models.
- **Done:** a manifest validates; a Ticket directory is parseable and registry-insertable.

### Phase B — Discovery & Registry
- Recursive scan for `*.ticket/`; build `registry.db` (derived SQLite).
- Relationship index from `metadata.json`.
- **Done:** rescan rebuilds an equivalent registry; dangling refs handled.

### Phase C — Watcher & Event Bus
- `inotify` watcher over repository → filesystem events.
- Adapter translating to domain events; in-process Event Bus with fan-out.
- **Done:** editing `metadata.json` emits `metadata.updated` to subscribers.

### Phase D — Scheduler, Dispatcher, Executor
- Per-workspace sequential queues; concurrent workspaces; ticket/action locks.
- `dispatcher.py` runner selection; `executor.py` + python/bash/node runners.
- Retry/timeout/debounce.
- **Done:** a hook runs on its event, under lock, with retries; rapid edits debounce.

### Phase E — Lifecycle & State
- `state.py` transitions enforcing Part VI rules; emit lifecycle events; persist state.json +
  activity.jsonl.
- **Done:** full `Created→…→Archived→Initialized` path enforced and logged.

### Phase F — Permissions
- Capability model + safe defaults; escalation prompts; executor enforcement.
- **Done:** a `network:true` request pauses for approval; denied capabilities are blocked.

### Phase G — CLI & SDK
- Typer CLI over `runtime.sock`; Python SDK client (shared `Runtime`).
- **Done:** `ticket runtime start`, `create`, `action`, `inspect` work end-to-end.

### Phase H — Hardening & Docs
- Crash recovery (stale lock reaping, registry repair), logging, tests, RFC extraction (Phase 2).
- **Done:** kill -9 runtime → restart recovers; test suite green.

## 5. Directory Layout
All implementation under `lib/ticket-management/` per earlier Parts.

## 6. Lifecycle
The roadmap itself is versioned; new phases appended as the framework grows (future runtimes).

## 7. Interaction
Each phase consumes the relevant Part(s): A→V, B→IV, C→II/VII, D→III/VIII, E→VI, F→IX, G→X/XI.

## 8. Failure Modes
| Risk | Mitigation |
| --- | --- |
| Watcher event loss | Debounce + periodic reconcile rescan (Phase C/D). |
| Lock deadlock | Acyclic dependency rule; boot reaping (Phase D/E). |
| Permission bypass | Enforcement at executor, not manifest (Phase F). |

## 9. Security
Security is built in from Phase F onward, not bolted on. Phases A–E operate on local files only.

## 10. Future Extensions (deferred, see Charter Non-Goals)
- Agent orchestration contract (delegation/handoff as first-class).
- AI provider integrations (local/Ollama/OpenRouter/MCP) behind providers.
- General plugin system + marketplace.
- Compiled/distributable Ticket packages.
- Windows/macOS support; distributed multi-node scheduling.
- Additional repositories: Skill, Workflow, Memory, Agent.

## 11. Examples
A contributor picking up Phase D implements the Scheduler and the three runners, then demonstrates a
Ticket whose `metadata.updated` hook runs under a ticket lock with a 2-retry policy.

## 12. Rationale
The order front-loads the riskiest, most architecture-defining components (watch → bus → schedule →
execute → lifecycle → permissions) so the system is demonstrable early and the later CLI/SDK are thin
wrappers. Explicit deferrals keep v1 scope honest and prevent the "ticket system" from silently
becoming an unbuildable mega-project. This roadmap is the bridge from the Master Specification to the
RFC suite (Phase 2), where each RFC versions independently.
