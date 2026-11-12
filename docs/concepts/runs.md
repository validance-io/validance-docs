# Runs

A run is one execution of a workflow. Every run is a separate, recorded event with its own identifier, evidence, and outcome.

## Triggering a run

```http
POST /api/workflows/{workflow_name}/trigger
Content-Type: application/json

{
  "parameters": {...}
}
```

Returns a run identifier immediately. The run executes asynchronously; you query it for status and results.

The run identifier is the address for everything the run produced: its record, its files, its logs, its variables, and its audit trail. In API paths and responses this identifier appears as `workflow_hash`.

## What a run records

The run record carries:

- The run identifier
- The workflow name
- The `definition_hash` of the definition that produced the run, and the definition version
- The status
- The parameter values used
- Start and end timestamps

Timestamps are ISO 8601 with microsecond precision. They carry no timezone offset and are written in the engine's own clock zone, which is UTC in a standard deployment.

For every task in the run, the record carries:

- A task identifier
- The task name
- The status
- Start and end timestamps, and a duration in seconds
- The container image used
- A resource-usage field, populated only where the deployment collects it
- An error message, when the task failed

This information is captured automatically. It is not opt-in, and turning it off is not a configuration option.

The run record is a summary. The full forensic detail for a task is carried in the audit trail, on the `task.completed` event, which records:

- The command that ran
- The container image, and the `sha256` digest of the image manifest that actually executed, so a mutable tag such as `:latest` still resolves to exact bytes after the fact
- The exit code
- `input_file_hashes` and `output_file_hashes`: a mapping of each path inside the container to its SHA-256 content hash
- The duration and the resource usage collected

The engine's own identity is recorded the same way, on `workflow.started`, as the digest of the engine image that ran rather than as a version string.

See [Audit and evidence](audit-and-evidence.md) for retrieval, and [Querying a run](#querying-a-run) for the endpoints that list files, logs and variables directly.

## Run states

| State | Meaning |
|---|---|
| `PENDING` | Accepted by the engine, not yet started |
| `RUNNING` | At least one task is executing |
| `SUCCESS` | All tasks completed successfully |
| `FAILED` | One or more tasks failed and the run cannot continue |
| `KILLED` | Explicitly cancelled |

State transitions are recorded in the audit trail.

!!! note "Compare status case-insensitively"

    The casing of the status string is not consistent across endpoints. The same run is reported as `FAILED` by the run list and as `failed` by the single-run endpoint, while the task entries inside that same response use `FAILED`. Clients should normalise case rather than compare against a literal.

## Querying a run

Run record, including the per-task entries:

```http
GET /api/runs/{workflow_hash}
```

Runs across the system, newest first, with `limit`, `offset`, `status`, `workflow_name`, `since` and `until` filters:

```http
GET /api/runs
```

If you know only the workflow name, its latest run is reachable through its status:

```http
GET /api/workflows/{workflow_name}/status
```

Files, with a SHA-256 `file_hash` for each, its size, the task it belongs to, and `is_input` / `is_output` flags:

```http
GET /api/files/{workflow_hash}
```

Captured standard output and standard error, per task:

```http
GET /api/logs/{workflow_hash}
```

Output variables each task wrote:

```http
GET /api/variables/{workflow_hash}
```

Audit trail for the run:

```http
GET /api/audit/{workflow_hash}
```

## Re-running a workflow

Triggering the same workflow again creates a new, independent run with a new identifier. Previous runs are not affected.

A run uses the revision currently active under the workflow's name, and the trigger call takes no `definition_hash`, so a revision cannot be selected for a single trigger. Running an earlier revision means making it active again, which is done by re-registering its definition under the same name: see [Running an earlier revision](workflows.md#running-an-earlier-revision).

What every run records is the `definition_hash` it executed, which is what establishes afterwards which revision produced which outcome.

## See also

- [Audit and evidence](audit-and-evidence.md) for the structure and verification of recorded evidence
- [Workflows](workflows.md) for workflow registration and identity
