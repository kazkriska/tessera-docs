# RFC-0008 — Permissions & Security Model

- **Status:** Ratified
- **Version:** 1.0
- **References:** Master Part IX; RFC-0000; RFC-0003; RFC-0004

## Summary
Capability model constraining what a hook/action may do, with human-approved escalation beyond safe
defaults. This is the security boundary; Tickets are untrusted, user/agent-editable code.

## Capability vocabulary
```yaml
permissions:
  filesystem:
    read:  [task/**, metadata.json]
    write: [state.json, activity.jsonl]
  network: false     # default false
  subprocess: true   # default true
  secrets: false     # default false (expose .env to script?)
```

## Safe defaults (no manifest permissions)
- `filesystem.read`: Ticket's own dir.
- `filesystem.write`: `state.json`, `activity.jsonl` only.
- `network: false`, `subprocess: true`, `secrets: false`.

## Escalation (human approval)
If a manifest requests more than defaults (e.g. `network:true`, write outside Ticket), the runtime
**holds** the task and requests approval via CLI prompt / GUI popup (future) / pre-authorized config.
Approval may be cached per Ticket or per capability. Denial → `ticket.permission.denied` in
`activity.jsonl`.

## Enforcement
The **Executor/runner** constructs a constrained environment (filtered env; future chroot/namespace)
before invoking. Pre-check occurs in Scheduler before hand-off.

## Failure modes
Unapproved capability → hold + prompt; on denial, skip+log. Script exceeds FS path → blocked write,
non-zero exit, logged. Network with `network:false` → socket blocked, logged.

## Rationale
Least-privilege-by-default with explicit escalation keeps Tickets portable and safe to share. Gating
at execution (not edit) time preserves "filesystem is source of truth" while protecting the host.
Data-shaped model allows future sandboxing without redesign.
