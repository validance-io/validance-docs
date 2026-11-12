# Tasks

A task is a single unit of work in a workflow. Every task runs in a container, with declared inputs and outputs.

## Task structure

| Field | Purpose |
|---|---|
| `name` | Identifier within the workflow |
| `command` | Command to execute inside the container |
| `docker_image` | Container image to run, by tag or by digest |
| `inputs` | Mapping of filename inside the container to its source |
| `output_files` | Mapping of variable name to the output filename the task writes |
| `output_vars` | Mapping of variable name to a type string |
| `depends_on` | Task names that must complete before this one |
| `environment` | Environment variables passed to the container, including secret references |
| `volumes` | Additional volume mounts |
| `parallel` | Whether the task may execute concurrently with other eligible tasks |
| `timeout` | Maximum execution time in seconds |
| `gate` | Approval tier required before the task runs. See [Approvals](#approvals) |
| `gate_timeout` | How long an approval hold may wait, in seconds |
| `token_budget` | Optional ceiling on tokens consumed by the task |
| `cost_budget` | Optional ceiling on cost incurred by the task |

Only `name` is mandatory. A task that omits `docker_image` inherits the workflow's default image.

An `outputs` field also exists and is deprecated. Use `output_files`.

## Container execution

Every task runs in its own container. The image is part of the task definition, and the image used is recorded on the task record for the run, so the image for a given execution can be identified later.

## Inputs

A task's inputs can be:

- **Files** staged into the container before execution
- **Parameters** passed at workflow trigger time
- **References to other task outputs**, using `@task_name:output_key` syntax

The engine resolves references at execution time and stages files into the container. Resolution is fail-closed: if a declared input cannot be resolved, the task does not execute and the run records why. A real failure message from the engine reads `task 'extract' declares 1 input(s) that could not be resolved; refusing to execute without them`.

## Outputs

A task declares its outputs explicitly. The engine:

- Records the SHA-256 content hash of every file it registers for the run, with the task it belongs to and whether it was an input or an output
- Captures the variables the task writes
- Validates that declared outputs are present, and fails the task if a declared output is missing

Implicit outputs, meaning files the task happens to leave behind without declaring them, are not part of the recorded evidence. The contract is the declaration.

## Resource constraints

Tasks can declare a timeout, and resource hints where the deployment supports them. The task record carries a `resource_usage` field, which is populated only when the deployment collects it.

## Secrets

A task references credentials by name, not by value, through its `environment` mapping. The engine resolves the reference at execution time and injects the value into the task's environment. The credential value never appears in the workflow definition or in the audit log; only the reference name is recorded.

## Approvals

A task declares its own approval requirement, through the `gate` field:

| Tier | Effect |
|---|---|
| `auto-approve` | The task runs without human involvement. This is the default |
| `human-confirm` | The engine creates a pending approval and holds the task until a human resolves it |
| `always-deny` | The task is never executed |

```python
Task(
    name="release_batch",
    command="python release.py",
    docker_image="qc-tools:2.1",
    gate="human-confirm",
    gate_timeout=3600,
)
```

`gate_timeout` bounds the hold, in seconds, and defaults to 300.

The tier is part of the workflow definition, so it is covered by the `definition_hash`: changing a task from `auto-approve` to `human-confirm` is a definition change and is detectable as one.

Enforcement is in the kernel, not in the caller, and it is fail-closed. The tier is validated against the closed set above when the task is constructed and again at execution time. A task that declares a tier the engine does not recognise, or whose gate cannot be made available, is failed with a categorised reason and **no container is ever started**. There is no execution path in which a declared gate is skipped.

While a task is held, the engine records the request on an `approval` entity in the audit trail, and records the resolution and its timestamp when it arrives. The declared tier is also recorded on the task's `task.started` event, so the evidence shows which gate applied to a step, not merely that a gate existed.

Pending approvals are listed and resolved through the API:

```http
GET  /api/approvals
POST /api/approvals/{approval_id}/resolve
```

## See also

- [Workflows](workflows.md) for how tasks compose into a graph
- [Runs](runs.md) for what happens at execution time
- [Audit and evidence](audit-and-evidence.md) for what is recorded
