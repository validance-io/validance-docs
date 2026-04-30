# Tasks

A task is a single unit of work in a workflow. Every task runs in a container, with declared inputs and outputs.

## Task structure

| Field | Purpose |
|---|---|
| `name` | Identifier within the workflow |
| `image` | Container image to run |
| `command` | Command to execute inside the container |
| `inputs` | Files, parameters, and references to outputs of other tasks |
| `output_files` | Files the task is expected to produce |
| `output_vars` | Typed values the task is expected to write |
| `depends_on` | Other tasks that must complete first |
| `timeout` | Optional execution timeout |
| `environment` | Environment variables, including secret references |

## Container execution

Every task runs in its own container. The container image is part of the task definition and is recorded on every run together with its content hash. The exact image used for a given run can always be identified later, including for replay and reproducibility.

## Inputs

A task's inputs can be:

- **Files** staged into the container before execution
- **Parameters** passed at workflow trigger time
- **References to other task outputs**, using `@task_name:output_key` syntax for variables and explicit paths for files

The engine resolves references at execution time, stages files into the container, and records the content hash of every input before the task runs.

## Outputs

A task declares its outputs explicitly. The engine:

- Records the content hash of every declared output file after the task completes
- Captures variables the task writes
- Validates that declared outputs are present; if a declared output is missing the run fails

Implicit outputs — files the task happens to leave behind without declaring — are not part of the recorded evidence. The contract is the declaration.

## Resource constraints

Tasks can declare resource hints (CPU, memory, timeout). Resource enforcement depends on deployment configuration.

## Approvals

A task can require human approval before it runs. When a task is gated, the engine pauses the workflow and creates a pending approval record; the workflow resumes only after the approval is resolved. The decision, the resolver, and the timestamp are recorded in the audit trail.

## Secrets

A task references credentials by name, not by value. The engine resolves the reference at execution time and injects the value into the task's environment. The credential value never appears in the workflow definition or in the audit log; only the reference name is recorded.

## See also

- [Workflows](workflows.md) for how tasks compose into a graph
- [Runs](runs.md) for what happens at execution time
- [Audit and evidence](audit-and-evidence.md) for what is recorded
