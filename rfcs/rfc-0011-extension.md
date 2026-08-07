# RFC-0011 — Extension System & Future Runtimes

- **Status:** Ratified
- **Version:** 1.0
- **References:** `foundation/charter.md` §8, §10; Master Part XII §10; RFC-0000; RFC-0001

## Summary
Tessera is a control plane for filesystem-native resources. The Ticket runtime is the first concrete
instance. This RFC defines the extension boundary and the path to additional resource repositories and
runtimes.

## Extension points (v1 minimal)
- **Runners** (`plugins/python_runner.py`, `bash_runner.py`, `node_runner.py`): add languages by
  adding a runner; no dispatcher change.
- **Manifest `permissions`** consume the capability model (RFC-0008).
- **Events** are open: any `name` can be emitted/subscribed.
- **Relationship types** are open in `metadata.json` (RFC-0002).

## Deferred extension system (future)
A general plugin system (storage providers, AI providers, event providers, CLI extensions, custom
manifest fields) is explicitly a future extension point, not a v1 requirement (Charter Non-Goals).
When added, it reuses the runner/permission/event substrate.

## Future resource repositories
The model (RFC-0002) is reused for:
```
FrameworkRoot/
├── TicketRepository/    (ticket-runtime — built first)
├── SkillRepository/     (skill-runtime — future)
├── WorkflowRepository/  (workflow-runtime — future)
├── MemoryRepository/    (memory-runtime — future)
└── lib/                 (shared substrate)
```
Each new resource type becomes its own RFC referencing RFC-0000/0001/0002, declaring only its
distinct behavior/manifest additions.

## Shared substrate
Event Bus, Scheduler, Runner/Permission/Registry, and SDK transport are reused across runtimes. The
framework does not hardcode "ticket" into every subsystem.

## Rationale
Keeping the framework resource-agnostic from day one avoids the most expensive late change: a rename
or redesign when new resource types appear. The Ticket remains the fundamental abstraction; other
resources decompose into Tickets.
