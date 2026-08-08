# Tessera — RFC Suite

**Status:** Derived from the stable Master Specification (Phase 1). Each RFC is independently
versioned and references the Master rather than duplicating it. When the Master and an RFC disagree,
the RFC (as the newer, versioned artifact) takes precedence for that subject; the Master is updated to
match in the next revision.

## Why RFCs
The user wanted a complete specification without losing detail to page-count reduction (Q15). The
compromise (ChatGPT's final proposal, accepted): write one coherent **Master** now, then extract each
major subject into a **versioned RFC** so future changes evolve one subject without silently editing
page 97 of a monolith.

## RFC index

| RFC | Title | Master source |
| --- | --- | --- |
| RFC-0000 | Philosophy & Guiding Principles | Foundation Charter §4, §6 |
| RFC-0001 | Architecture & Component Pipeline | Master Part II |
| RFC-0002 | Ticket Model & Resource Abstraction | Master Part IV |
| RFC-0003 | Manifest Specification (v1) | Master Part V |
| RFC-0004 | Runtime Internals | Master Part III |
| RFC-0005 | Event Bus | Master Part VII |
| RFC-0006 | Scheduler, Queues & Locking | Master Part VIII |
| RFC-0007 | Ticket Lifecycle & State Machine | Master Part VI |
| RFC-0008 | Permissions & Security Model | Master Part IX |
| RFC-0009 | Command-Line Interface | Master Part X |
| RFC-0010 | SDK Architecture | Master Part XI |
| RFC-0011 | Extension System & Future Runtimes | Charter §8, Master Part XII §10 |
| RFC-0012 | Implementation Roadmap | Master Part XII |
| RFC-0013 | Distribution & Installation | Phase 2 — RFCs (authoritative distribution spec) |

## Conventions
- Each RFC carries: `Status`, `Version`, `Summary`, normative sections, and `References` (Master Part
  + other RFCs).
- RFCs are mutable via `Version` bumps; the Master is the historical "book."
- New resource types (Skill, Workflow, Memory) will become their own RFCs referencing RFC-0000/0001/0002.
