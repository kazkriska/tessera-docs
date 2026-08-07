# RFC-0002 — Ticket Model & Resource Abstraction

- **Status:** Ratified
- **Version:** 1.0
- **References:** Master Part IV; RFC-0000 (I-3); RFC-0003; RFC-0007

## Summary
The Ticket is the fundamental resource abstraction. A Ticket is a portable, self-contained directory
that carries data, configuration, state, history, and behavior. Other concepts (Skill, Workflow,
Memory, Agent) decompose into atomic units that are themselves Tickets.

## On-disk structure
```
HQ_BR-001.ticket/
├── MANIFEST.yaml      # behavior (source of behavior)
├── metadata.json      # descriptive metadata + identity + relationships
├── state.json         # current runtime state
├── activity.jsonl     # append-only history (one JSON object per line)
├── task/  (assets/)   # work artifacts
├── scripts/           # reusable scripts
├── hooks/             # event-reactive code (manifest is authoritative)
└── .env               # execution environment (overrides global)
```

## Identity (I-3)
- One immutable `id` = directory basename minus `.ticket`. Stored in `metadata.json`.
- Title is mutable. Relationships and logs key on `id`.

## Relationships (graph model)
`metadata.json` MAY declare: `parent`, `children` (mirror), `depends_on`, `blocks`, `duplicates`,
`references`, `related_to`, `spawned_from`, `delegated_to`. Parent/child is one relationship type;
the runtime builds a relationship index. No directory nesting required.

## Workspace reference
A Ticket points to an external Workspace via `.env` (`WORKSPACE=`) or `metadata.json`. The repository
is not the workspace.

## Failure modes
Missing MANIFEST → not registered. `id`≠basename → rejected. Missing state.json → reinit. Corrupt
metadata → excluded. Dangling relationship → ignored with warning.

## Rationale
Separating behavior/metadata/state/history/env keeps the source of truth clean and tooling simple.
The graph model outgrows a pure tree for workflow/dependency needs.
