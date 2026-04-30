# Runs

A run is one execution of a workflow. Every run is a separate, recorded event with its own identifier, evidence, and outcome.

## Triggering a run

```http
POST /api/workflows/{name}/trigger
Content-Type: application/json

{
  "parameters": {...}
}
```

Returns a run identifier immediately. The run executes asynchronously; you query it for status and results.

## What a run records

Every run is associated with:

- A unique run identifier
- The workflow definition hash at the time of trigger
- The engine version
- The triggering user
- A start timestamp (UTC, microsecond precision)
- The parameter values used

For every task in the run:

- Container image and image hash
- Command executed
- Input file content hashes
- Output file content hashes
- Output variable values
- Exit code
- Standard output and standard error
- Timing
- Host that executed it

This information is captured automatically. It is not opt-in, and turning it off is not a configuration option.

## Run states

| State | Meaning |
|---|---|
| `pending` | Accepted by the engine, not yet started |
| `running` | At least one task is executing |
| `success` | All tasks completed successfully |
| `failed` | One or more tasks failed and the run cannot continue |
| `killed` | Explicitly cancelled by the user |

State transitions are recorded in the audit trail.

## Querying a run

Status and metadata:

```http
GET /api/runs/{run_id}
```

Per-task evidence:

```http
GET /api/runs/{run_id}/tasks
GET /api/runs/{run_id}/tasks/{task_name}
```

Files produced:

```http
GET /api/runs/{run_id}/files
```

Lineage (input-step-output graph):

```http
GET /api/runs/{run_id}/lineage
```

Audit trail for the run:

```http
GET /api/runs/{run_id}/audit
```

## Re-running a workflow

Triggering the same workflow again creates a new, independent run with a new identifier. Previous runs are not affected.

To reproduce an earlier run with the exact workflow definition, trigger by definition hash rather than by name:

```http
POST /api/workflows/by-hash/{definition_hash}/trigger
```

This pins the run to a specific workflow version even if the workflow has been updated since.

## See also

- [Audit and evidence](audit-and-evidence.md) for the structure and verification of recorded evidence
- [Workflows](workflows.md) for workflow registration and identity
