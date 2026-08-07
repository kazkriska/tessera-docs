# Part I — Vision

## 1. Purpose
State the long-term intent of Tessera so that every later design decision can be checked against it.
Vision is not implementation; it is the gravity well that keeps the architecture coherent as it grows.

## 2. Motivation
The user began with a concrete need: a self-contained directory that reacts to changes in itself and
in peer directories, executing automation that changes its own state or another directory's state.
The exploration revealed this need is not "ticketing" — it is *filesystem-native automation* where
the directory is both data and program.

## 3. Responsibilities
- Define the problem Tessera solves in one sentence.
- Establish the "resource abstraction" thesis: Tickets are the fundamental unit; other concepts
  decompose into Tickets.
- Set the multi-runtime horizon: Ticket runtime is the first of a family.

## 4. Design
### The one-sentence vision
> **Tessera gives the filesystem a lifecycle.** Each resource is a self-contained, portable tile;
> the runtime watches the filesystem, translates changes into events, and executes declared
> automation — so directories become living, reactive units rather than passive folders.

### The resource abstraction thesis
A Ticket is not a project-management artifact. It is:
- portable and self-contained,
- identified by an immutable id,
- carrying metadata, state, history, configuration, and automation,
- linked to an external *Workspace* where work is performed,
- with a lifecycle and an event/hook/action contract.

Anything the framework might manage later — Skill, Workflow, Memory, Prompt, Agent, Dataset — is
expressible as a Ticket distinguished only by `metadata.json` and declared behavior. The runtime
never needs to know the semantic type. This is why the project is named for a *tile in a mosaic*:
each resource composes into a larger automation picture, and the metaphor holds regardless of how
many resource types are added.

### The control-plane thesis
Tessera is a control plane for filesystem-native resources. The Ticket runtime is the first concrete
instance of a generic engine (Discovery → Registry → Watcher → Event Bus → Scheduler → Dispatcher →
Executor). Future runtimes (workflow, skill, agent) share the substrate.

## 5. Directory Layout
Not applicable to Vision. (See Master index for layout.)

## 6. Lifecycle
Not applicable to Vision.

## 7. Interaction
Vision constrains Architecture (Part II), which realizes it through components.

## 8. Failure Modes
None at the Vision level. Contradicting the Vision is a design failure caught in review.

## 9. Security
The vision assumes resources are user/agent-editable (Option A). Security (Part IX) must preserve
that editability while constraining what automation may do.

## 10. Future Extensions
- Additional resource repositories (Skill, Workflow, Memory, Agent) under `FrameworkRoot/`.
- Multiple runtime processes cooperating across repositories (distributed execution, currently
  deferred — see Charter Non-Goals).

## 11. Examples
A `FeatureRequest.ticket`, a `Memory.ticket`, and a `Skill.ticket` are structurally identical. The
runtime processes them uniformly; only their manifests and metadata differ.

## 12. Rationale
Naming the project after the tile metaphor (tessera) intentionally resists "Ticket System" framing.
It keeps the door open for the framework to grow beyond ticketing without a rename or redesign, which
is the single most expensive change to make late. The Vision is recorded first so that growth
decisions have a fixed reference point.
