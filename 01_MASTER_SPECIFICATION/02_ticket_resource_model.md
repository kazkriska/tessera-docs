# FRAME Master Architecture Specification
## Chapter 2: Ticket Resource Model & Directory Anatomy

---

## 1. Ticket Directory Bundle Anatomy

A **Ticket** in FRAME is represented on disk as a self-contained directory bundle whose name ends with the suffix `.ticket` (or standard folder containing `MANIFEST.yaml`).

```
HQ_BR-scope-20260807_001.ticket/
├── MANIFEST.yaml            # Declarative contract (schema, hooks, actions, exports)
├── metadata.json            # Identity & domain metadata (static / structural)
├── state.json               # Dynamic runtime state & lifecycle status
├── activity.jsonl           # Append-only structured execution & audit log
├── .env                     # Local environment variables & secrets (dotenv format)
├── scripts/                 # Execution scripts & event handlers
│   ├── setup_workspace.py
│   ├── on_metadata_changed.py
│   └── delegate_agent.py
└── task/                    # Working payload directory
    ├── task-id.json
    └── assets/              # Output artifacts, documents, images, data files
        ├── report.docx
        ├── summary.csv
        └── diagram.png
```

---

## 2. File Specifications & Standard Schemas

### 2.1 `MANIFEST.yaml` (Declarative Contract)
Defines identity, API version, required runtime environments, event watch mappings, hooks, actions, and exported parameters. *(Detailed in Chapter 3).*

### 2.2 `metadata.json` (Identity & Domain Data)
Contains structured, domain-specific information about the Ticket resource. It describes **what the resource is**, who created it, its scope, and its classification.

#### `metadata.json` Schema (JSON Schema v7)
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "FrameTicketMetadata",
  "type": "object",
  "required": ["id", "title", "kind", "created_at", "owner"],
  "properties": {
    "id": {
      "type": "string",
      "pattern": "^[A-Za-z0-9_-]+$",
      "description": "Unique identifier for the ticket across the workspace."
    },
    "title": {
      "type": "string",
      "description": "Human-readable summary title."
    },
    "kind": {
      "type": "string",
      "default": "ticket",
      "description": "Resource classification (e.g., ticket, task, skill, workflow, memory)."
    },
    "scope": {
      "type": "string",
      "description": "Organizational or project domain (e.g., HQ_BR, FRONTEND, DATA_PIPE)."
    },
    "version": {
      "type": "string",
      "default": "1.0.0"
    },
    "created_at": {
      "type": "string",
      "format": "date-time"
    },
    "owner": {
      "type": "object",
      "required": ["name", "type"],
      "properties": {
        "name": { "type": "string" },
        "type": { "type": "string", "enum": ["user", "agent", "system"] },
        "email": { "type": "string" }
      }
    },
    "tags": {
      "type": "array",
      "items": { "type": "string" }
    },
    "custom": {
      "type": "object",
      "description": "Arbitrary domain-specific metadata key-values."
    }
  }
}
```

---

### 2.3 `state.json` (Dynamic Runtime State)
Stores volatile execution state, current lifecycle status, assignee details, parent/child relationships, and progress tracking.

#### `state.json` Schema (JSON Schema v7)
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "FrameTicketState",
  "type": "object",
  "required": ["status", "updated_at", "step"],
  "properties": {
    "status": {
      "type": "string",
      "enum": ["created", "initializing", "ready", "running", "blocked", "handoff", "completed", "failed", "archived"],
      "description": "Current lifecycle status of the ticket."
    },
    "step": {
      "type": "string",
      "description": "Current execution milestone or active pipeline step."
    },
    "updated_at": {
      "type": "string",
      "format": "date-time"
    },
    "assignee": {
      "type": "object",
      "properties": {
        "id": { "type": "string" },
        "type": { "type": "string", "enum": ["human", "agent", "service"] },
        "assigned_at": { "type": "string", "format": "date-time" }
      }
    },
    "relationships": {
      "type": "object",
      "properties": {
        "parent_id": { "type": "string" },
        "parent_path": { "type": "string" },
        "children": {
          "type": "array",
          "items": {
            "type": "object",
            "required": ["id", "path"],
            "properties": {
              "id": { "type": "string" },
              "path": { "type": "string" },
              "status": { "type": "string" }
            }
          }
        },
        "dependencies": {
          "type": "array",
          "items": { "type": "string" }
        }
      }
    },
    "last_execution": {
      "type": "object",
      "properties": {
        "action": { "type": "string" },
        "exit_code": { "type": "integer" },
        "timestamp": { "type": "string", "format": "date-time" },
        "trace_id": { "type": "string" }
      }
    },
    "variables": {
      "type": "object",
      "description": "Runtime variables populated by hook handlers."
    }
  }
}
```

---

### 2.4 `activity.jsonl` (Append-Only Event & Audit Log)
Every action, hook invocation, state transition, or file trigger is recorded as an immutable JSON line in `activity.jsonl`.

#### Sample `activity.jsonl` Entries
```json
{"timestamp":"2026-08-07T19:15:00Z","trace_id":"trc-98124","event":"ticket.created","actor":{"id":"user_kaz","type":"human"},"details":{"kind":"ticket"}}
{"timestamp":"2026-08-07T19:15:02Z","trace_id":"trc-98124","event":"hook.invoked","hook":"metadata.updated","entrypoint":"scripts/on_metadata_changed.py","status":"running"}
{"timestamp":"2026-08-07T19:15:04Z","trace_id":"trc-98124","event":"state.transition","from":"initializing","to":"ready","actor":{"id":"script","type":"system"},"details":{"exit_code":0}}
```

---

## 3. Storage Integrity & Atomic File Operations

To prevent race conditions, partial reads, or file corruption when multiple processes or agents access a Ticket, FRAME enforces **Atomic Disk Write Protocols**.

### 3.1 Atomic JSON Writes (`state.json` & `metadata.json`)
Directly overwriting JSON files (`open('state.json', 'w')`) is strictly forbidden. All state mutations MUST follow the **Write-Temp-Rename** pattern:

```python
import json
import os
import tempfile

def atomic_write_json(filepath: str, data: dict):
    dir_name = os.path.dirname(filepath)
    # 1. Create temporary file in the target directory
    with tempfile.NamedTemporaryFile('w', dir=dir_name, delete=False, encoding='utf-8') as tf:
        json.dump(data, tf, indent=2)
        temp_name = tf.name
    
    # 2. Atomically replace the target file (POSIX atomic operation)
    os.replace(temp_name, filepath)
```

### 3.2 Concurrent Log Appends (`activity.jsonl`)
Appends to `activity.jsonl` MUST use advisory file locking (`fcntl.flock` on Linux/macOS) to prevent line interleaving during concurrent subprocess execution:

```python
import fcntl
import json

def append_activity_log(log_path: str, record: dict):
    line = json.dumps(record) + "\n"
    with open(log_path, 'a', encoding='utf-8') as f:
        fcntl.flock(f.fileno(), fcntl.LOCK_EX)
        try:
            f.write(line)
            f.flush()
        finally:
            fcntl.flock(f.fileno(), fcntl.LOCK_UN)
```

---

## 4. File Specification Rationale Matrix

| File | Format | Primary Consumer | Rationale |
| :--- | :--- | :--- | :--- |
| `MANIFEST.yaml` | YAML 1.2 | FRAME Runtime / Humans | Supports comments, hierarchical structure, and clean declarative syntax for handlers and triggers. |
| `metadata.json` | JSON | Systems / Scripts | Strict structured schema; fast parsing without external YAML dependencies in lightweight scripts. |
| `state.json` | JSON | FRAME Runtime / SDK | Frequent write access; atomic replace support (`os.replace`) ensures corrupt-free state updates. |
| `activity.jsonl` | JSON Lines | Log Aggregators / Audit | Append-only format; lines can be processed individually via `tail -f`, `grep`, or streaming parsers. |
| `.env` | Key=Value | Process Runners | Industry standard for local shell and subprocess environment variable injections. |
