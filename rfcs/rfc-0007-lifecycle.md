# RFC-0007 — Ticket Lifecycle & State Machine

- **Status:** Ratified
- **Version:** 1.0
- **References:** Master Part VI; RFC-0000; RFC-0002; RFC-0005; RFC-0006

## Summary
Eight fixed lifecycle states with constrained (but flexible) transitions. States are fixed; the order
may vary (a Ticket may re-block after delegation, or archive then re-initialize).

## The eight states
1. **Created** — discovered/registered. Always first.
2. **Initialized** — `initialize` ran. Must be followed by Ready.
3. **Ready** — prepared. May follow Blocked/Archived/Delegated.
4. **Running** — active. Must follow Ready or Blocked.
5. **Blocked** — stuck. Any stage except Completed/Archived.
6. **Delegated** — handed to agent/sub-process. Must follow Ready or Initialized.
7. **Completed** — finished. Only exit is Archived.
8. **Archived** — retired. Renews only via Initialized (re-init).

## Transition rules (machine-checkable)
| From→To | Allowed |
| --- | --- |
| (none)→Created | ✅ |
| Created→Initialized→Ready | ✅ (required chain) |
| Ready→{Running,Delegated,Blocked} | ✅ |
| Blocked→{Ready,Running} | ✅ |
| Running→{Blocked,Delegated,Completed} | ✅ |
| Delegated→{Blocked,Ready,Completed} | ✅ |
| Completed→Archived | ✅ only |
| Archived→Initialized | ✅ renew |
| Completed→Ready | ❌ (archive first) |
| Archived→Running | ❌ (re-init first) |
| Blocked→Completed | ❌ |
| (any)→Created | ❌ (entry-only) |

## Lifecycle events
Runtime emits `ticket.created/initialized/ready/running/blocked/delegated/completed/archived` and
`ticket.reinitialized` on Archived→Initialized. Hooks subscribe like any domain event.

## Persistence
Current state in `state.json` (`{"lifecycle":"Running", ...}`). Every transition appends a line to
`activity.jsonl` (`{"ts":...,"event":"ticket.running","from":"Ready","to":"Running"}`).

## Failure modes
Illegal transition → rejected, logged, `ticket.transition.rejected` emitted. Missing/corrupt
state.json → assume Created, re-init. Concurrent transition → serialized by ticket lock.

## Rationale
Fixed states give automation a stable vocabulary; flexible transitions respect real-world messiness.
Encoding the user's exact Q10 rules as a table makes the contract enforceable and testable.
`Archived→Initialized` (never directly to Running) prevents zombie tickets skipping setup.
