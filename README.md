# FRAME: Filesystem Resource Automation & Management Engine
## Master Architectural Specification & Design Suite

> **Status:** Draft / Initial Master Specification  
> **Version:** 1.0.0-draft  
> **Source of Truth:** Filesystem (`*.ticket` Directory Bundles)

---

## Executive Summary

**FRAME** (*Filesystem Resource Automation & Management Engine*) is a filesystem-native, resource-agnostic automation mesh and workflow engine.

In FRAME, the **filesystem is the primary source of truth**, and the **Ticket is the fundamental resource abstraction**. A Ticket is not merely an issue tracker item; it is a self-contained, active directory object encapsulating state, metadata, scripts, assets, hooks, and declarative manifests (`MANIFEST.yaml`).

The FRAME Runtime acts as a lightweight daemon and controller—interpreting manifests, observing filesystem changes, translating low-level OS events into high-level domain events, and executing action/hook pipelines in isolated execution environments.

---

## Specification Sitemap

| File / Document | Description | Status |
| :--- | :--- | :--- |
| [Project Charter & Vision](file:///home/kaz/NewProjects/UnnamedFileSystemAutomationProject/docs/00_CHARTER_AND_VISION.md) | Framework vision, philosophy, invariants, non-goals, naming rationale, and core principles. | **Complete** |
| [01. Architecture Overview](file:///home/kaz/NewProjects/UnnamedFileSystemAutomationProject/docs/01_MASTER_SPECIFICATION/01_architecture_overview.md) | Layered architecture, workspace discovery, ephemeral cache model, component interactions. | **Complete** |
| [02. Ticket Resource Model](file:///home/kaz/NewProjects/UnnamedFileSystemAutomationProject/docs/01_MASTER_SPECIFICATION/02_ticket_resource_model.md) | Standard `*.ticket` directory structure, metadata, state schemas, append-only event logs. | **Complete** |
| [03. Manifest Specification](file:///home/kaz/NewProjects/UnnamedFileSystemAutomationProject/docs/01_MASTER_SPECIFICATION/03_manifest_specification.md) | `MANIFEST.yaml` schema, API versioning (`apiVersion: ticket/v1`), hooks, actions, and capabilities. | **Complete** |
| [04. Runtime Engine Internals](file:///home/kaz/NewProjects/UnnamedFileSystemAutomationProject/docs/01_MASTER_SPECIFICATION/04_runtime_engine_internals.md) | Ephemeral index, event debouncing, priority queuing, locking, and crash recovery. | **Complete** |
| [05. Execution & Sandboxing](file:///home/kaz/NewProjects/UnnamedFileSystemAutomationProject/docs/01_MASTER_SPECIFICATION/05_execution_and_sandboxing.md) | POSIX subprocess isolation, environment variable masking, `uv` integration, process groups. | **Complete** |
| [06. Lifecycle & State Machine](file:///home/kaz/NewProjects/UnnamedFileSystemAutomationProject/docs/01_MASTER_SPECIFICATION/06_lifecycle_and_state_machine.md) | Formal state machine states, transitions, handoffs, and parent-child dependency graphs. | **Complete** |
| [07. CLI & SDK Specification](file:///home/kaz/NewProjects/UnnamedFileSystemAutomationProject/docs/01_MASTER_SPECIFICATION/07_cli_and_sdk_specification.md) | Command-line interface (`frame` CLI), Python SDK (`frame-sdk`), and agent IPC interface. | **Complete** |
| [08. Implementation Roadmap](file:///home/kaz/NewProjects/UnnamedFileSystemAutomationProject/docs/01_MASTER_SPECIFICATION/08_implementation_roadmap.md) | Phased engineering milestones from core SDK prototype to RFC extraction and verification. | **Complete** |

---

## Standalone RFC Suite (`docs/rfcs/`)

Derived directly from the Master Specification for public open-source publication and protocol standards tracking:

| RFC Document | Title / Scope | Status |
| :--- | :--- | :--- |
| [RFC-0000](file:///home/kaz/NewProjects/UnnamedFileSystemAutomationProject/docs/rfcs/RFC-0000_FRAME_Philosophy_and_Guiding_Principles.md) | **FRAME Philosophy, Vision, and Invariants** | **Active** |
| [RFC-0001](file:///home/kaz/NewProjects/UnnamedFileSystemAutomationProject/docs/rfcs/RFC-0001_Filesystem_Native_Layered_Architecture.md) | **Layered System Topology and Workspace Discovery** | **Active** |
| [RFC-0002](file:///home/kaz/NewProjects/UnnamedFileSystemAutomationProject/docs/rfcs/RFC-0002_Ticket_Resource_Bundle_Specification.md) | **Ticket Directory Bundle Anatomy, Schemas, and Atomic Writes** | **Active** |
| [RFC-0003](file:///home/kaz/NewProjects/UnnamedFileSystemAutomationProject/docs/rfcs/RFC-0003_Declarative_Manifest_Format.md) | **MANIFEST.yaml Declarative Schema v1** | **Active** |
| [RFC-0004](file:///home/kaz/NewProjects/UnnamedFileSystemAutomationProject/docs/rfcs/RFC-0004_Runtime_Engine_Event_Bus_and_Debouncing.md) | **Event Engine, Debouncing Mechanics, and Priority Scheduler** | **Active** |
| [RFC-0005](file:///home/kaz/NewProjects/UnnamedFileSystemAutomationProject/docs/rfcs/RFC-0005_Subprocess_Execution_and_Sandboxing.md) | **POSIX Subprocess Execution, Path Jailing, and Env Masking** | **Active** |
| [RFC-0006](file:///home/kaz/NewProjects/UnnamedFileSystemAutomationProject/docs/rfcs/RFC-0006_State_Machine_and_Graph_Dependency_Resolution.md) | **Ticket Lifecycle State Machine and Dependency Graph Unblocking** | **Active** |
| [RFC-0007](file:///home/kaz/NewProjects/UnnamedFileSystemAutomationProject/docs/rfcs/RFC-0007_CLI_and_Python_SDK_Interface.md) | **`frame` Command-Line Interface and `frame-sdk` Library Contract** | **Active** |
| [RFC-0008](file:///home/kaz/NewProjects/UnnamedFileSystemAutomationProject/docs/rfcs/RFC-0008_Implementation_Roadmap_and_Verification.md) | **Phased Implementation Milestones and Quality Verification Strategy** | **Active** |

---

## Key Guiding Principles

1. **Filesystem Sovereignty**: All application data, state, logs, and configuration live in portable human-readable files on disk. No hidden databases host primary state.
2. **Resource Agnosticism**: Everything (Tasks, Workflows, Skills, AI Agent Handoffs, Memories) is modeled as a Ticket resource with custom metadata and hooks.
3. **Decoupled Controller Architecture**: Tickets declare *what* happens (`MANIFEST.yaml`); the Runtime daemon handles *how* events are watched and executed.
4. **Ephemeral Epiphenomenality**: Databases, indexes, and in-memory graphs are disposable read caches. Destroying the runtime cache causes zero data loss.
5. **Polyglot & Extensible**: Action and hook handlers are standard scripts (Python via `uv`, Bash, Node.js) executed via clean subprocess contracts.
