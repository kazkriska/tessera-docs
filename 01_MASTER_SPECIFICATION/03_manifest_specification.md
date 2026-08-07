# FRAME Master Architecture Specification
## Chapter 3: Manifest Specification (`MANIFEST.yaml`)

---

## 1. Overview

The `MANIFEST.yaml` file is the **declarative contract** of a Ticket resource. It informs the FRAME Runtime of the Ticket's identity, execution dependencies, low-level file watches, domain event subscriptions (`hooks`), named capabilities (`actions`), and security boundaries.

---

## 2. Complete `MANIFEST.yaml` Schema (`apiVersion: ticket/v1`)

Below is the formal Kubernetes-inspired structure for FRAME manifests.

```yaml
apiVersion: ticket/v1
kind: Ticket

metadata:
  id: HQ_BR-scope-20260807_001
  name: "Infographic Generation & Report Task"
  version: "1.0.0"
  description: "Automated data processing, infographic creation, and agent handoff."
  tags:
    - analytics
    - automated-report
    - agent-ready

spec:
  # 1. Required Execution Environments
  runtime:
    python: ">=3.12"
    toolchain: uv

  # 2. Local Environment Variables & Defaults
  env:
    LOG_LEVEL: INFO
    MAX_RETRY_COUNT: "3"
    OUTPUT_FORMAT: png

  # 3. File System Watch Mappings (Low-level -> Hook)
  watch:
    - path: "metadata.json"
      events: [modify]
      trigger: metadata_updated

    - path: "task/assets/**"
      events: [create, modify]
      trigger: asset_indexed

    - path: ".env"
      events: [modify]
      trigger: env_reloaded

  # 4. Domain Event Hooks (Event -> Handlers)
  hooks:
    ticket.initialized:
      - entrypoint: scripts/setup_workspace.py
        timeout: 60
        async: false

    metadata_updated:
      - entrypoint: scripts/on_metadata_changed.py
        timeout: 30
        async: true

    asset_indexed:
      - entrypoint: scripts/index_assets.py
        timeout: 45
        async: true

    parent.changed:
      - entrypoint: scripts/refresh_context.py
        timeout: 30

  # 5. Named Callable Actions (Exposed APIs)
  actions:
    configure-assignee:
      description: "Assigns task to human or AI agent with role context."
      entrypoint: scripts/configure_assignee.py
      timeout: 30
      retry: 2
      permissions:
        - read:ticket
        - write:state
      inputs:
        assignee_id:
          type: string
          required: true
        assignee_type:
          type: string
          enum: [human, agent]
          required: true
      outputs:
        status:
          type: string

    delegate-subagent:
      description: "Hand off task bundle to a specialized subagent."
      entrypoint: scripts/delegate_to_subagent.py
      timeout: 300
      retry: 1
      permissions:
        - read:ticket
        - write:state
        - create:child_ticket

  # 6. Exported Output Variables
  exports:
    - task.status
    - task.report_path
    - task.assignee
```

---

## 3. Detailed Field Specifications

### 3.1 `apiVersion` & `kind`
- `apiVersion`: Must be `ticket/v1` for the current specification. Enables backwards compatibility as the engine evolves.
- `kind`: Specifies the resource classification. Default is `Ticket`. (Future extension kinds include `Workflow`, `Skill`, `Agent`).

### 3.2 `spec.runtime`
Declares system requirements needed to run the scripts:
- `python`: Version specifier string (e.g. `>=3.10`, `>=3.12`).
- `toolchain`: Execution helper (e.g. `uv`, `venv`, `system`).

### 3.3 `spec.watch`
Maps low-level filesystem events to named triggers:
- `path`: Glob pattern relative to Ticket root (e.g. `metadata.json`, `task/assets/**`).
- `events`: Array of watched OS file events: `create`, `modify`, `delete`, `move`.
- `trigger`: Internal event name emitted to the event bus upon match.

### 3.4 Uniform Handler Object Spec (`hooks` & `actions`)
Both `hooks` and `actions` utilize a uniform handler object schema for complete predictability across the framework:

| Key | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `entrypoint` | `string` | *(Required)* | Relative path to execution script (e.g., `scripts/setup.py`, `scripts/deploy.sh`). |
| `timeout` | `integer` | `30` | Maximum execution time in seconds before process is forcibly SIGKILLed. |
| `retry` | `integer` | `0` | Number of automatic retries on non-zero process exit codes. |
| `async` | `boolean` | `true` | If `true`, runs in background worker thread; if `false`, blocks queue. |
| `permissions` | `list[string]`| `[]` | Explicit scope permissions granted to the executing script. |

---

## 4. Manifest Validation Rules

When the FRAME Runtime discovers a `MANIFEST.yaml`, it performs strict syntax & semantic validation before registering the Ticket:
1. **Entrypoint Existence**: Every `entrypoint` path declared under `hooks` or `actions` MUST exist on disk inside the Ticket directory.
2. **Glob Pattern Validity**: All `watch.path` globs must be valid POSIX glob expressions.
3. **Circular Watch Guard**: A `watch` rule MUST NOT watch `state.json` or `activity.jsonl` if its target hook modifies those files, preventing immediate infinite recursion.

---

## 5. Rationale Matrix

| Decision | WHAT | WHY | HOW |
| :--- | :--- | :--- | :--- |
| **Uniform Handler Schema** | Use identical keys (`entrypoint`, `timeout`, `permissions`) for both hooks and actions. | Eliminates code duplication in the runtime dispatcher; simplifies manifest parsing logic. | Dispatcher processes both hook objects and action objects through a single `ExecutionSpec` data model. |
| **Kubernetes-style Manifest** | Include `apiVersion` and `kind` top-level keys. | Allows future framework growth (e.g. adding `kind: Workflow` or `apiVersion: ticket/v2`) without breaking older runtimes. | Parser inspects `apiVersion` first to route YAML to the appropriate validator schema. |
| **Explicit Input/Output Schemas** | Declare `inputs` and `outputs` for exposed `actions`. | Allows AI agents and CLI callers to introspect callable capabilities cleanly without inspecting python source code. | Exposed via `frame inspect <ticket_id>` CLI and SDK interfaces. |
