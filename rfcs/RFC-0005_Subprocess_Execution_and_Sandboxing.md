# RFC-0005: Subprocess Execution and Sandboxing

- **RFC Number:** RFC-0005
- **Title:** POSIX Subprocess Execution, Path Jailing, and Environment Masking
- **Status:** Draft / Active
- **Author:** FRAME Core Architecture Team
- **Created:** 2026-08-07
- **Replaces:** None
- **Category:** Standards Track / Execution

---

## 1. Abstract

This RFC specifies the execution contract for polyglot script handlers inside FRAME. To maintain a lightweight footprint without requiring container daemons, execution relies on POSIX subprocess isolation, process group control, environment variable masking, and Python `uv` toolchain resolution.

---

## 2. Process Group Termination & Timeouts

Every hook or action handler is executed inside a new POSIX process group (`os.setsid`).
- When execution exceeds its declared `timeout` (default: 30s), the runtime sends `SIGTERM` to the process group (`pgid`), followed by `SIGKILL` after 3s if children linger.
- This guarantees zero orphaned background processes when scripts time out.

---

## 3. Environment Variable Masking

Before launching a subprocess, sensitive host environment variables (e.g. `AWS_SECRET_ACCESS_KEY`, `DATABASE_URL`) are masked unless explicitly whitelisted. Environment variables merge in order:
`Host Base Env` $\rightarrow$ `Workspace .env` $\rightarrow$ `Ticket .env` $\rightarrow$ `Manifest Spec Env` $\rightarrow$ `Event STDIN Payload`.

---

## 4. STDIN Payload Ingestion

Context data is passed to script handlers on STDIN as a single JSON payload:
```json
{
  "event": "ticket.metadata.updated",
  "ticket_id": "HQ_BR-001",
  "timestamp": "2026-08-07T19:45:00Z",
  "trace_id": "trc-12345"
}
```

---

## 5. References & Related RFCs

- [RFC-0003: Declarative Manifest Format](file:///home/kaz/NewProjects/UnnamedFileSystemAutomationProject/docs/rfcs/RFC-0003_Declarative_Manifest_Format.md)
