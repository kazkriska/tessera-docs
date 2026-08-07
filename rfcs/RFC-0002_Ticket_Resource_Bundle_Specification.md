# RFC-0002: Ticket Resource Bundle Specification

- **RFC Number:** RFC-0002
- **Title:** Ticket Directory Bundle Anatomy, Schemas, and Atomic Write Protocol
- **Status:** Draft / Active
- **Author:** FRAME Core Architecture Team
- **Created:** 2026-08-07
- **Replaces:** None
- **Category:** Standards Track / Data Format

---

## 1. Abstract

This RFC standardizes the directory bundle structure for FRAME Tickets (`*.ticket/`), detailing file roles, normative JSON schemas for `metadata.json` and `state.json`, append-only audit logging (`activity.jsonl`), and POSIX atomic write requirements (`os.replace` and `flock`).

---

## 2. Directory Layout

```
HQ_BR-scope-20260807_001.ticket/
├── MANIFEST.yaml            # Declarative contract
├── metadata.json            # Identity & domain metadata
├── state.json               # Dynamic runtime state
├── activity.jsonl           # Immutable JSONL audit log
├── .env                     # Local environment variables
├── scripts/                 # Execution scripts & event handlers
└── task/
    └── assets/              # Output files, images, documents
```

---

## 3. Normative Schemas

### 3.1 `metadata.json` (Identity Data)
MUST contain: `id` (string), `title` (string), `kind` (string), `created_at` (ISO-8601 string), `owner` (object with `name` and `type`).

### 3.2 `state.json` (Runtime State)
MUST contain: `status` (enum: `created`, `initializing`, `ready`, `running`, `blocked`, `handoff`, `completed`, `failed`, `archived`), `updated_at` (ISO-8601 string), `step` (string).

### 3.3 `activity.jsonl` (Audit Log)
Append-only log where each line is a valid JSON object containing `timestamp`, `trace_id`, `event`, `actor`, and optional `details`.

---

## 4. Storage Integrity Protocols

### 4.1 Atomic JSON Writes
All mutations to `state.json` or `metadata.json` MUST use Write-Temp-Rename (`os.replace`):
1. Write JSON data to a temporary file in the same directory (`.state.json.tmp`).
2. Execute atomic POSIX rename (`os.replace`) to overwrite the target JSON file.

### 4.2 Concurrent Log Locking
Appends to `activity.jsonl` MUST acquire POSIX advisory file locks (`fcntl.flock(fd, LOCK_EX)`) on `activity.jsonl.lock` prior to writing.

---

## 5. References & Related RFCs

- [RFC-0003: Declarative Manifest Format](file:///home/kaz/NewProjects/UnnamedFileSystemAutomationProject/docs/rfcs/RFC-0003_Declarative_Manifest_Format.md)
