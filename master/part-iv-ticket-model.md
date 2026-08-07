# Part IV — Ticket Model

## 1. Purpose
Define the Ticket as the fundamental resource: its on-disk shape, identity, the files it contains,
and the relationships it can declare. This is the data contract every other Part depends on.

## 2. Motivation
The original request was a "self-contained directory" that reacts to change. Concretely that directory
is a Ticket. We must specify exactly what files it contains, what each is for, how identity works, and
how Tickets relate to one another — without overloading the term "workspace."

## 3. Responsibilities
- Provide a stable, portable directory format for a resource.
- Separate data, configuration, state, history, and behavior (Charter §philosophy).
- Assign immutable identity and mutable presentation (title).
- Express relationships as a graph, not just a tree.

## 4. Design

### 4.1 On-disk structure
```
HQ_BR-001.ticket/
├── MANIFEST.yaml      # declarative behavior (source of behavior)
├── metadata.json      # structured metadata (source of descriptive facts)
├── state.json         # mutable runtime state (source of current status)
├── activity.jsonl     # append-only history (source of record)
├── task/              # work artifacts produced/consumed
│   └── assets/        # images, documents, csv, docx, etc.
├── scripts/           # reusable scripts invoked by hooks/actions
├── hooks/             # event-reactive code (optional; manifest is authoritative)
└── .env               # execution environment (overrides global)
```

### 4.2 File responsibilities
| File | Format | Source of… | Safe to delete? |
| --- | --- | --- | --- |
| `MANIFEST.yaml` | YAML | behavior | No — defines the Ticket's automation |
| `metadata.json` | JSON | descriptive metadata (id, title, type, owner, relationships) | No — identity/reference data |
| `state.json` | JSON | current runtime state (lifecycle state, assignee, locks, computed) | No — current status; runtime may reinit if missing |
| `activity.jsonl` | JSONL | historical record (one JSON object per line) | No — audit trail |
| `task/` | arbitrary | work artifacts | No — user/agent data |
| `scripts/`, `hooks/` | code | implementation | No — behavior implementation |
| `.env` | dotenv | execution environment | No — configuration |

### 4.3 Identity (Invariant I-3)
- Every Ticket has exactly one **immutable `id`** (e.g. `HQ_BR-001`). It is stored in `metadata.json`
  and must match the directory basename minus the `.ticket` suffix. Renaming the directory requires
  updating `metadata.json` consistently; the id itself never changes.
- **Title** is mutable and lives in `metadata.json`. Relationships and logs key on `id`, never title.
- Rationale: stable references survive renames; logs and relationship graphs stay valid.

### 4.4 Relationships (graph model)
The user chose a **graph**, not just parent/child. `metadata.json` MAY declare:
- `parent` (single, optional) — tree nesting for convenience.
- `children` (derived/optional) — convenience mirror of others' `parent`.
- `depends_on: [ids]`
- `blocks: [ids]`
- `duplicates: [ids]`
- `references: [ids]`
- `related_to: [ids]`
- `spawned_from: [id]`
- `delegated_to: [id]` (agent/assignment target)

Parent/child is merely one relationship type. The runtime builds a relationship index from these
fields; no directory nesting is required (Tickets are independent files/dirs at the repository root).

### 4.5 Workspace reference
A Ticket points to an **external Workspace** (where work is performed) via `.env` (`WORKSPACE=/path`)
or `metadata.json`. The repository is NOT the workspace. The runtime does not discover Tickets inside
a workspace; it operates there only under the Ticket's granted permissions.

## 5. Directory Layout
Covered in §4.1. The repository root holds many such Ticket directories side by side; the runtime
indexes them.

## 6. Lifecycle
See Part VI. The Ticket's current lifecycle state lives in `state.json`.

## 7. Interaction
- **Discovery** reads `metadata.json` + `MANIFEST.yaml` to register the Ticket.
- **Event Bus** carries events scoped to the repository and `lib/ticket-management/` (per user answer
  to Q15).
- **Hooks/Actions** read/write `state.json` and `activity.jsonl` under lock.
- **Relationships** are consulted by the Scheduler (dependency ordering) and by parent/child event
  propagation.

## 8. Failure Modes
| Failure | Handling |
| --- | --- |
| Missing `MANIFEST.yaml` | Ticket not registered for automation; flagged. |
| `id` ≠ directory basename | Registration rejected; operator must reconcile. |
| Missing `state.json` | Runtime initializes default state (Created/Initialized) on next lifecycle pass. |
| Corrupt `metadata.json` | Ticket excluded; error logged; quarantine optional. |
| Dangling relationship id | Relationship ignored with warning; does not block the Ticket. |

## 9. Security
Tickets are user/agent-editable (Option A). The runtime never needs write access to `MANIFEST.yaml`
or `metadata.json` to function; it writes only `state.json`, `activity.jsonl`, and files a
hook/action creates under its granted permissions (Part IX).

## 10. Future Extensions
- **Compiled packages:** a future tool may package a Ticket directory into a single signed artifact
  for distribution (deferred; directory remains source of truth).
- **Additional resource repositories** (Skill, Workflow, Memory) reuse this exact model.

## 11. Examples
```json
// HQ_BR-001.ticket/metadata.json
{
  "id": "HQ_BR-001",
  "title": "Onboard new contributor",
  "type": "task",
  "owner": "alice",
  "parent": null,
  "depends_on": ["HQ_BR-000"],
  "blocks": [],
  "references": ["SKILL-onboarding"],
  "workspace": "/home/alice/work/onboarding"
}
```

## 12. Rationale
The separation of `MANIFEST` (behavior), `metadata` (facts), `state` (status), `activity` (history),
and `.env` (environment) is the single most important structural decision. It lets tools, UIs, and
the runtime each read what they need without parsing scripts, and it keeps the source of truth
(filesystem) cleanly distinguished from derived runtime state. The graph relationship model future-
proofs the design against workflow/dependency needs that a pure tree cannot express.

---

> **REVISION — Appended from FRAME documentation (Rev A · 2026-08-07).**
> *Non-destructive: all prior text in this document is unchanged and remains canonical. This block augments it with material drawn from the FRAME spec set (same design, independent authorship). Status: Appended.*

### R.A.4 — Formal schemas & atomic-write protocol (from FRAME Ch.2)

Tessera Part IV describes file responsibilities in prose; FRAME supplies machine-checkable JSON Schemas and the write protocol. Appended for implementers.

**`metadata.json` (JSON Schema v7):**

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "TesseraTicketMetadata",
  "type": "object",
  "required": ["id", "title", "kind", "created_at", "owner"],
  "properties": {
    "id": {"type": "string", "pattern": "^[A-Za-z0-9_-]+$"},
    "title": {"type": "string"},
    "kind": {"type": "string", "default": "ticket"},
    "scope": {"type": "string"},
    "version": {"type": "string", "default": "1.0.0"},
    "created_at": {"type": "string", "format": "date-time"},
    "owner": {"type": "object", "required": ["name", "type"],
      "properties": {"name": {"type": "string"},
        "type": {"type": "string", "enum": ["user", "agent", "system"]},
        "email": {"type": "string"}}},
    "tags": {"type": "array", "items": {"type": "string"}},
    "custom": {"type": "object"}
  }
}
```

**`state.json` (JSON Schema v7):**

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "TesseraTicketState",
  "type": "object",
  "required": ["status", "updated_at", "step"],
  "properties": {
    "status": {"type": "string",
      "enum": ["created","initializing","ready","running","blocked","handoff","completed","failed","archived"]},
    "step": {"type": "string"},
    "updated_at": {"type": "string", "format": "date-time"},
    "assignee": {"type": "object",
      "properties": {"id": {"type": "string"},
        "type": {"type": "string", "enum": ["human","agent","service"]},
        "assigned_at": {"type": "string", "format": "date-time"}}},
    "relationships": {"type": "object",
      "properties": {
        "parent_id": {"type": "string"},
        "children": {"type": "array", "items": {"type": "object",
          "required": ["id","path"],
          "properties": {"id": {"type": "string"}, "path": {"type": "string"}, "status": {"type": "string"}}}},
        "dependencies": {"type": "array", "items": {"type": "string"}}}},
    "last_execution": {"type": "object",
      "properties": {"action": {"type": "string"}, "exit_code": {"type": "integer"},
        "timestamp": {"type": "string", "format": "date-time"}, "trace_id": {"type": "string"}}},
    "variables": {"type": "object"}
  }
}
```

**Atomic JSON writes (`state.json`, `metadata.json`).** Direct overwrite is forbidden; use write-temp + `os.replace`:

```python
import json, os, tempfile
def atomic_write_json(filepath, data):
    d = os.path.dirname(filepath)
    with tempfile.NamedTemporaryFile('w', dir=d, delete=False, encoding='utf-8') as tf:
        json.dump(data, tf, indent=2)
        tmp = tf.name
    os.replace(tmp, filepath)   # POSIX-atomic
```

**Concurrent log appends (`activity.jsonl`).** Use advisory locking to prevent line interleaving:

```python
import fcntl, json
def append_activity_log(path, record):
    line = json.dumps(record) + "\n"
    with open(path, 'a', encoding='utf-8') as f:
        fcntl.flock(f.fileno(), fcntl.LOCK_EX)
        try:
            f.write(line); f.flush()
        finally:
            fcntl.flock(f.fileno(), fcntl.LOCK_UN)
```
