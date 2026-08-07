# Tessera — Filesystem-Native Resource Automation Framework

> **Spec status:** Phase 0 (Foundation) and Phase 1 (Master Specification) complete.
> Phase 2 (RFC suite) derived from the stable master.
> This tree is the authoritative design source of truth for the project, written to be implemented by engineers who were not in the room when the decisions were made.

## What Tessera is

Tessera is a **filesystem-native, resource-agnostic automation framework**. Its fundamental
resource is the **Ticket**: a portable, self-contained directory that carries its own data,
metadata, state, history, configuration, and automation. A long-running runtime daemon
discovers Tickets, watches the filesystem for changes, translates those changes into domain
events, and dispatches automation through a Scheduler, Dispatcher, and language-agnostic
Executors.

A Ticket is *not* a project-management ticket. It is a resource abstraction: any other
concept the framework might later manage — a Skill, a Workflow, a Memory, a Prompt, an Agent,
a Dataset — decomposes into atomic units that are themselves Tickets. The runtime does not need
to know the semantic meaning of a Ticket; it manages only the lifecycle and automation contract.

Even the name is chosen for this: a **tessera** is a single tile in a mosaic. Each Ticket is a
self-contained tile; together they compose a larger picture of automation. The metaphor survives
even as new resource types are added.

## Document map

| Path | Document | Phase |
| --- | --- | --- |
| `foundation/charter.md` | Project Charter & Architecture Vision | Phase 0 — Foundation |
| `master/index.md` | Master Specification — index & conventions | Phase 1 — Master |
| `master/part-i-vision.md` | Part I — Vision | Phase 1 |
| `master/part-ii-architecture.md` | Part II — Architecture | Phase 1 |
| `master/part-iii-runtime.md` | Part III — Runtime Internals | Phase 1 |
| `master/part-iv-ticket-model.md` | Part IV — Ticket Model | Phase 1 |
| `master/part-v-manifest.md` | Part V — Manifest Specification (v1) | Phase 1 |
| `master/part-vi-lifecycle.md` | Part VI — Lifecycle & State Machine | Phase 1 |
| `master/part-vii-event-bus.md` | Part VII — Event Bus | Phase 1 |
| `master/part-viii-scheduler.md` | Part VIII — Scheduler | Phase 1 |
| `master/part-ix-permissions.md` | Part IX — Permissions & Security | Phase 1 |
| `master/part-x-cli.md` | Part X — Command-Line Interface | Phase 1 |
| `master/part-xi-sdk.md` | Part XI — SDK Architecture | Phase 1 |
| `master/part-xii-roadmap.md` | Part XII — Implementation Roadmap | Phase 1 |
| `rfcs/README.md` | RFC suite index | Phase 2 — RFCs |
| `rfcs/rfc-*.md` | RFC-0000 … RFC-0012 | Phase 2 — RFCs |

## How to read this

1. Start with `foundation/charter.md`. Every later decision references it.
2. Read `master/index.md` for the section template, terminology, and invariants.
3. Read the Parts in any order after that; each Part is self-contained but cross-references others.
4. For versioned, independently-evolving specs, consult the matching RFC (Phase 2).

## Source of truth

- **Filesystem is the source of truth.** The Tickets in `TicketRepository/` are canonical and
  editable (Option A). The runtime interprets them; it does not own them.
- **Runtime state is disposable.** `.ticket-runtime/` (registry.db, locks, cache, logs, sockets)
  can be deleted at any time and rebuilt by rescanning.
- **Manifests are purely declarative.** Logic lives only in hooks/actions; the manifest describes.

See `foundation/charter.md` §Guiding Principles for the full list.
