# RFC-0009 — Command-Line Interface

- **Status:** Ratified
- **Version:** 1.0
- **References:** Master Part X; RFC-0004; RFC-0010

## Summary
Official CLI built with **Typer** (`cli.py`); a thin client over the runtime's own functions, talking
to `runtime.sock`. Basic surface defined in v1; extensibility left for later.

## Command surface (v1)
```
ticket runtime start [--repo PATH]   # boot or attach to existing socket
ticket runtime stop                  # graceful shutdown
ticket runtime status                # running? queue depths? tickets?
ticket repo init [PATH]              # scaffold repository + .ticket-runtime/
ticket repo scan                     # force rediscovery + registry rebuild
ticket create <id> [--type TYPE]     # scaffold .ticket + minimal manifest
ticket inspect <id>                  # metadata/state/manifest summary
ticket transition <id> <state>       # request validated lifecycle transition
ticket action <id> <action> [args]   # invoke named action (may prompt)
ticket validate <id>                 # validate manifest vs schema
ticket log <id> [--tail N]           # tail activity.jsonl
```

## Invocation model
- `runtime start` creates/attaches `runtime.sock`. A second `start` attaches and reports "already
  running" rather than spawning a competing watcher (I-6).
- `action`/`transition` send requests to the running runtime (validation, permission checks,
  scheduling).

## Failure modes
Runtime not running → non-destructive commands run on FS; mutating ones error. Invalid transition →
rejected with reason. Permission denied → shown + recorded.

## Rationale
A small, predictable CLI built directly on runtime functions keeps the human interface consistent with
the SDK and avoids a second behavior implementation. Defining it in v1 gives operators a way to drive
and debug immediately.
