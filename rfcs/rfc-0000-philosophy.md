# RFC-0000 — Philosophy & Guiding Principles

- **Status:** Ratified
- **Version:** 1.0
- **References:** `foundation/charter.md` §4, §6; Master `index.md`

## Summary
Establishes the non-negotiable philosophy and invariants of Tessera. Every other RFC must conform.

## Philosophy (10 principles)
1. Filesystem is the source of truth.
2. Tickets are portable and self-contained.
3. Runtime is stateless and rebuildable.
4. Manifests are declarative.
5. Runtime state is disposable.
6. Everything is event-driven.
7. Extensibility over hardcoding.
8. Language-agnostic hook execution.
9. Explicit configuration over convention.
10. Single runtime per workspace (TicketRepository).

## Invariants (must always hold)
- **I-1** Runtime modifies the repository only via explicitly declared, locked state mutations.
- **I-2** Deleting `.ticket-runtime/` never loses Ticket data or corrupts a Ticket.
- **I-3** Every Ticket has one immutable `id`; titles may change.
- **I-4** Events are notifications; return values use actions.
- **I-5** A hook triggers further automation only by emitting an event.
- **I-6** Exactly one runtime process owns a given TicketRepository.
- **I-7** The manifest is data; no conditional/loop logic.
- **I-8** Locks acquired only around state mutations (ticket + action level).
- **I-9** Rescanning always reconstructs an equivalent registry.

## Non-Goals (v1)
Agent orchestration as first-class contract; built-in AI; full plugin marketplace; compiled Ticket
packages; Windows/macOS; cross-machine distributed execution. (See Charter §7.)

## Rationale
These principles resolve naming and architectural ambiguity before code exists. They are the contract
that lets the framework grow into multiple runtimes without a redesign.
