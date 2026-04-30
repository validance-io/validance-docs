# Workflows

A workflow in Validance is a JSON document describing a set of tasks, the dependencies between them, and the inputs and outputs that flow through the graph. You register a workflow with the engine and trigger runs of it by name.

## What a workflow contains

- A name (its identifier in the catalog)
- A version
- A list of tasks, each with its own definition (see [Tasks](tasks.md))
- The dependency relationships between tasks

## Authoring

A workflow can be authored two ways:

- **As JSON directly**, useful for non-Python clients, generated workflows, and system-to-system integration.
- **Using the Python SDK** (`validance-sdk` on PyPI, Apache 2.0), which provides type-checked task and workflow construction. The SDK serialises to the same JSON format.

Both produce the same artefact: a JSON workflow definition that you register with the engine.

## Identity

Every workflow has a `definition_hash`: a SHA-256 over the canonical JSON form of its definition. The hash is the workflow's identity — it changes if and only if the definition changes.

This means:

- Two workflows with the same hash are byte-equivalent definitions
- Updating a workflow produces a new hash
- "What ran on this date" is an exact, content-addressable answer
- Change detection between two definitions reduces to hash comparison

The hash is recorded with every run. Audit records reference the hash, not the workflow name, so the linkage from a recorded outcome back to the exact definition that produced it is unambiguous.

## Registering a workflow

```http
POST /api/workflows
Content-Type: application/json

{
  "name": "data.processing",
  "version": "1.0",
  "tasks": [...]
}
```

The engine validates the JSON against its schema and stores it. If validation fails the engine returns the validation errors and rejects the workflow. The response includes the assigned `definition_hash`.

## Updating a workflow

Updating a workflow registers a new version with a new hash. Previous runs remain associated with their original definition; the update does not affect them retroactively.

## Triggering a run

```http
POST /api/workflows/{name}/trigger
Content-Type: application/json

{
  "parameters": {...}
}
```

Returns a run identifier. The run executes asynchronously — see [Runs](runs.md) for status, evidence, and querying.

To pin a run to an exact prior version of a workflow, trigger by hash instead of by name:

```http
POST /api/workflows/by-hash/{definition_hash}/trigger
```

## See also

- [Tasks](tasks.md) for the structure of an individual task
- [Runs](runs.md) for what happens when a workflow executes
- [Audit and evidence](audit-and-evidence.md) for what is recorded
