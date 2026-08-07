# RFC-0003 — Manifest Specification (v1)

- **Status:** Ratified
- **Version:** 1.0
- **References:** Master Part V; RFC-0000 (I-7); RFC-0002; RFC-0008

## Summary
`MANIFEST.yaml` is the purely declarative, single source of behavior. No conditional/loop logic
(Invariant I-7). Logic lives only in hooks/actions.

## Envelope
```yaml
apiVersion: ticket/v1
kind: Ticket
metadata:
  id: HQ_BR-001
  title: "..."
  type: task
```

## Field grammar (v1)
```yaml
runtime:
  python: ">=3.12"
initialize:
  - run: scripts/setup.py
    shell: python
hooks:
  metadata.updated:
    - run: scripts/update_summary.py
      shell: python
      timeout: 30
      retry: 2
      async: false
actions:
  delegate:
    run: scripts/delegate.py
    shell: python
    timeout: 300
    retry: 3
    permissions: [write:ticket, create:child]
    inputs:  { assignee: {type: string} }
    outputs: { ticket_id: {type: string} }
permissions:
  filesystem: { read: [task/**], write: [state.json, activity.jsonl] }
  network: false
  subprocess: true
env:
  inherit: true
```

## Uniform executable descriptor
Both hooks and actions use the same shape (`run`, `shell`, `timeout`, `retry`, `async`). The only
semantic difference: a hook is *triggered by an event*; an action is *invoked by name*.

## Rules
- `shell` selects a runner; unknown `shell` rejected unless a matching runner plugin exists.
- `.env` resolution: Global `Tickets/.env` → Ticket `.env` (later overrides). `env.inherit:false`
  disables global.
- YAML anchors/aliases/merge keys **forbidden** by the loader (treated as error).
- `metadata.id` must equal directory basename.

## Rationale
Kubernetes-style `apiVersion`/`kind` enables format evolution. Uniform descriptors avoid special-
casing. Forbidding logic/anchors keeps the manifest validatable and predictable — the whole point of
"explicit over convention."
