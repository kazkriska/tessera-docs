# RFC-0004 — Runtime Internals

- **Status:** Ratified
- **Version:** 1.0
- **References:** Master Part III; RFC-0001; RFC-0005; RFC-0006; RFC-0008

## Summary
The runtime is a single Python 3.12+ daemon (via `uv`) owning one TicketRepository. It implements the
pipeline stages as focused modules.

## Module layout (`lib/ticket-management/`)
```
runtime/
  watcher.py     # inotify over repository
  dispatcher.py  # runner selection
  manifest.py    # load + validate MANIFEST.yaml
  executor.py    # run via a runner
  state.py       # lifecycle + state.json
  env.py         # .env resolution
plugins/
  python_runner.py / bash_runner.py / node_runner.py
cli.py           # Typer CLI
pyproject.toml   # uv, python >=3.12
```

## Boot sequence
1. Load `config.yaml`.
2. Singleton: if `runtime.sock` responds, attach (don't watch); else create socket (+ fallback pid).
3. Open/repair `registry.db` (derived).
4. Discovery: scan `*.ticket/`, validate manifests, register, build relationship index.
5. Start Watcher (inotify).
6. Run loop: events → bus → scheduler → dispatcher → executor; serve socket.
7. On signal: drain, release locks, remove socket/pid.

## Wiring
Watcher → adapter → Event Bus → Scheduler → Dispatcher → Executor → Ticket (under lock).

## Environment resolution (`env.py`)
Global `Tickets/.env` → Ticket `.env` (later overrides). Manifest `env.inherit:false` suppresses
global. Secrets never written to `activity.jsonl`.

## Failure modes
Corrupt registry → recreate+rescan. Watcher drop → debounce+reconcile. Runner crash → captured,
retried. Stale socket → reaped on boot. Unhandled hook exception → isolated, runtime continues.

## Rationale
Focused modules honor "extensibility over hardcoding" and keep each stage replaceable. Boot order
makes the registry rebuildable and singleton enforceable (I-2, I-6, I-9). `uv` provides reproducible
Python without making `venv` the behavior layer.

---

> **REVISION — Layout rename (Rev B · 2026-08-08).**
> *Non-destructive: prior text unchanged, remains canonical. The package tree moved to a src/ layout: `lib/ticket-management/` → `src/tessera_runtime/`; `tessera/` (SDK) → `src/tessera_sdk/`. Imports: `from tessera import …` → `from tessera_sdk import …`; `ticket_management.cli:main` → `tessera_sdk.cli:main`. Status: Appended.*

### R.B.1 — Affected references in this document

The rows below map canonical text above (left) to its Rev B equivalent (right). The canonical text is **not** rewritten; read it through this table.

| Location (canonical text) | As written (Rev A, canonical) | Rev B equivalent |
|---|---|---|
| § Module layout | `lib/ticket-management/` | `src/tessera_runtime/` |
| § Module layout | imports `lib.ticket_management.*` | `tessera_runtime.*` |

No canonical sentence above is amended by this block; tooling that resolves paths or imports MUST apply the mapping table. Rev A text remains the authority on *behaviour*; Rev B is the authority on *location*.
