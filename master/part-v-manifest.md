# Part V — Manifest Specification (v1)

## 1. Purpose
Specify `MANIFEST.yaml`: the purely declarative contract that tells the runtime *what* automation a
Ticket has. This is the single source of behavior (Charter principle 4 & 9). Schema is defined here;
versioning and full field enumeration are detailed so implementers need nothing else.

## 2. Motivation
Early designs considered discovering behavior from file names (`scripts/on_metadata_changed.py`).
That was rejected: implicit conventions are hard to validate, version, and introspect. The manifest
is explicit (principle 9). It must be declarative only — no conditionals/loops (Invariant I-7).
Logic belongs in hooks/actions.

## 3. Responsibilities
- Declare the Ticket's `apiVersion`, `kind`, and identity metadata.
- Declare `initialize` steps (lifecycle boot).
- Declare `hooks` (event → handlers).
- Declare `actions` (named, callable capabilities with metadata).
- Declare `permissions`, `relationships` references, and execution metadata (timeouts, retries, run mode).
- Declare environment inheritance and runtime requirements.

## 4. Design

### 4.1 Envelope (Kubernetes-style versioning)
```yaml
apiVersion: ticket/v1
kind: Ticket
metadata:
  id: HQ_BR-001          # MUST equal directory basename minus .ticket
  title: "Onboard contributor"
  type: task
```
The runtime rejects manifests without a recognized `apiVersion`/`kind`. Future formats use
`ticket/v2`, etc.

### 4.2 Field grammar (v1)
```yaml
apiVersion: ticket/v1
kind: Ticket
metadata:
  id: <string, immutable>
  title: <string>
  type: <string>

runtime:
  python: ">=3.12"        # optional requirement declaration

initialize:               # run once on lifecycle Initialized
  - run: scripts/setup_workspace.py
    shell: python

hooks:                    # event -> ordered handlers
  metadata.updated:
    - run: scripts/update_summary.py
      shell: python
      timeout: 30
      retry: 2
      async: false
  parent.changed:
    - run: scripts/sync_context.py

actions:                  # named, callable capabilities
  delegate:
    run: scripts/delegate_to_subagent.py
    shell: python
    timeout: 300
    retry: 3
    permissions:
      - write:ticket
      - create:child
    inputs:
      assignee: { type: string }
      priority: { type: integer }
    outputs:
      ticket_id: { type: string }

permissions:              # default capability requests (see Part IX)
  filesystem:
    read:  [task/**]
    write: [state.json, activity.jsonl]
  network: false
  subprocess: true

env:
  inherit: true           # inherit global .env (default true)
```

### 4.3 Uniform executable descriptor
Both `hooks` entries and `actions` use the same object shape:
```yaml
run: <path>
shell: <python|bash|node|...>   # defaults per config; runner chosen by Dispatcher
timeout: <seconds>
retry: <int>
async: <bool>
```
This uniformity means the runtime has one parsing path (user agreed: make hooks and actions
consistent). The only semantic difference: a hook is *triggered by an event*; an action is
*invoked by name* (by user, hook-via-event, agent, or CLI/SDK).

### 4.4 `shell` / runner resolution
`shell` selects a runner. The runtime ships `python`, `bash`, `node` runners. Unknown `shell`
values are rejected at validation unless a matching runner plugin is installed.

### 4.5 Inheritance
- `.env` resolution: Global `Tickets/.env` → Ticket `.env` (later overrides earlier). Manifest `env.inherit: false` disables global inheritance.
- Relationship/env propagation is explicit, not implicit.

## 5. Directory Layout
Manifest lives at the Ticket root as `MANIFEST.yaml`. It references paths under `scripts/`, `hooks/`.

## 6. Lifecycle
`initialize` runs during the `Initialized` state. Hooks fire on their events. Actions run on
invocation. (See Part VI.)

## 7. Interaction
- **Manifest Loader** (`lib/ticket-management/runtime/manifest.py`) parses and validates.
- **Registry** stores manifest version + parsed summary.
- **Scheduler/Dispatcher** consume hooks/actions.
- **Permission system** consumes `permissions`.

## 8. Failure Modes
| Failure | Handling |
| --- | --- |
| Unknown `apiVersion` | Reject Ticket; do not register. |
| `metadata.id` ≠ dir basename | Reject. |
| Missing `run` in a hook/action | Validation error; Ticket flagged. |
| Unknown `shell` | Validation warning/error (no runner). |
| YAML anchors/aliases/merge keys | Forbidden by the loader (treat as error) to avoid hidden behavior. |

## 9. Security
The manifest cannot execute code by itself. It only *references* scripts that run under the
Permission system. Escalation (e.g. `network: true` when default is false) requires human approval
(Part IX). The manifest is data; it is never `eval`'d.

## 10. Future Extensions
- **Expressions:** deferred permanently per user decision (logic stays in hooks). If ever added, it
  would be a new `apiVersion`, not a patch to v1.
- **Lua DSL manifest:** considered; rejected for v1 in favor of YAML's static analyzability.
- **Custom manifest fields via plugins:** future plugin system may extend schema; v1 keeps schema closed.

## 11. Examples
See §4.2 for a complete v1 example. A minimal valid manifest:
```yaml
apiVersion: ticket/v1
kind: Ticket
metadata:
  id: HQ_BR-002
  title: "Minimal ticket"
hooks:
  metadata.updated:
    - run: scripts/index.py
      shell: python
```

## 12. Rationale
Kubernetes-style `apiVersion`/`kind` lets the format evolve without breaking old Tickets. The uniform
executable descriptor avoids special-casing hooks vs actions. Forbidding YAML anchors/aliases and
logic keeps the manifest *predictable and validatable* — the entire point of "explicit over
convention." YAML (not TOML) was chosen because the manifest is hierarchical behavior (hooks,
actions, lifecycle, permissions), where TOML becomes awkward (user agreed after comparison).
