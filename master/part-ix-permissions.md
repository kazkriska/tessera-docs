# Part IX — Permissions & Security

## 1. Purpose
Define the capability model that constrains what a hook or action may do, and the escalation path
that requires human approval before a Ticket gains powers beyond safe defaults.

## 2. Motivation
Hooks and actions execute code on the user's machine and may touch external workspaces, networks, and
subprocesses. Without a permission model, a Ticket's automation could exfiltrate data or corrupt
files. The user required: standard safe defaults; overrides only after human approval (prompt, CLI
input, popup) or sudo (user Q11).

## 3. Responsibilities
- Provide default-deny / least-privilege baseline capabilities.
- Let manifests *request* capabilities (filesystem paths, network, subprocess).
- Enforce granted capabilities at execution time (in the runner/executor).
- Gate escalation behind explicit human approval.
- Keep the model open to future sandboxing (e.g. container/seccomp) without changing the contract.

## 4. Design

### 4.1 Capability vocabulary
```yaml
permissions:
  filesystem:
    read:  [task/**, metadata.json]
    write: [state.json, activity.jsonl]
  network: false          # default false
  subprocess: true        # default true (for scripts that shell out)
  secrets: false          # may the runner expose .env secrets to the script?
```
- **filesystem:** path-scoped read/write. Wildcards allowed (`task/**`).
- **network:** boolean. `true` enables outbound network from the runner.
- **subprocess:** boolean. `true` allows the script to spawn processes.
- **secrets:** boolean. `true` injects `.env` values into the environment; default `false` (scripts
  receive only non-secret env unless explicitly granted).

### 4.2 Defaults
Safe baseline applied when a manifest declares no `permissions`:
- `filesystem.read`: Ticket's own directory.
- `filesystem.write`: `state.json`, `activity.jsonl` only.
- `network: false`
- `subprocess: true` (scripts commonly shell out; can be tightened)
- `secrets: false`

### 4.3 Escalation (human approval)
If a manifest requests a capability stricter than the default (e.g. `network: true`, or write outside
the Ticket), the runtime **pauses** and requests approval via:
- interactive CLI prompt,
- GUI popup (future),
- or pre-authorized `sudo`/config entry.

Until approved, the task is held (not executed). Approval may be persisted per Ticket or per capability
so repeated runs don't re-prompt. Denial records `ticket.permission.denied` in `activity.jsonl`.

### 4.4 Enforcement point
The **Executor/runner** enforces capabilities: it constructs a constrained environment (filtered env,
optional chroot/namespace in future) before invoking the script. The Permission check happens in the
Scheduler/Dispatch path *before* hand-off.

## 5. Directory Layout
No dedicated on-disk artifact beyond manifest `permissions:` and approval cache under
`.ticket-runtime/cache/approvals/`.

## 6. Lifecycle
Permissions are evaluated at execution time for every hook/action invocation.

## 7. Interaction
- **Manifest** declares requested capabilities.
- **Scheduler/Dispatcher** pre-check against granted set.
- **Executor** enforces at runtime.
- **CLI/SDK** surfaces approval prompts.

## 8. Failure Modes
| Failure | Handling |
| --- | --- |
| Unapproved capability requested | Hold task; prompt; on denial, record + skip. |
| Script exceeds granted FS path | Runner blocks write; non-zero exit; logged. |
| Network attempted with `network:false` | Runner blocks socket; logged. |

## 9. Security
This is the security boundary. Because Tickets are user/agent-editable (Option A), the runtime must
treat manifest-declared behavior as untrusted code and gate its power. Secrets are never written to
`activity.jsonl`. Locks (Part VIII) prevent concurrent unsafe mutation.

## 10. Future Extensions
- **Sandboxing:** run hooks in a container/namespace; seccomp profiles per capability.
- **Signed manifests:** verify authorship before granting elevated capabilities.
- **Policy files:** repository-wide capability policy in `config.yaml`.

## 11. Examples
A Ticket's manifest requests `network: true` to call an API. On first run the runtime prompts: "Allow
HQ_BR-001 network access? [y/N]". User approves; approval cached. Subsequent runs proceed without
prompt. Revocation clears the cache entry.

## 12. Rationale
Least-privilege-by-default with explicit escalation keeps Tickets portable and safe to share: a Ticket
from elsewhere won't silently gain network or filesystem power. Gating at execution (not at edit) time
preserves the "filesystem is source of truth / user-editable" principle while still protecting the
host. The model is intentionally data-shaped so future sandboxing can be layered without redesign.
