# Workflows

A workflow in Validance is a JSON document describing a set of tasks, the dependencies between them, and the inputs and outputs that flow through the graph. You register a workflow with the engine and trigger runs of it by name.

## What a workflow contains

- A name, which is its identifier in the catalog
- A version
- An optional description
- A list of tasks, each with its own definition (see [Tasks](tasks.md))

Dependencies are not a separate structure. Each task names the tasks it waits on in its own `depends_on` field, and the graph is the union of those declarations.

## Authoring

A workflow can be authored two ways:

- **As JSON directly**, useful for non-Python clients, generated workflows, and system-to-system integration.
- **Using the Python SDK** (`validance-sdk` on PyPI, Apache 2.0), which provides type-checked task and workflow construction. The SDK serialises to the same JSON format.

Both produce the same artefact: a JSON workflow definition that you register with the engine.

## Identity

Every registered workflow has a `definition_hash`: a SHA-256 over the canonical JSON form of its definition. The hash is the workflow's identity, and it changes if and only if the definition changes.

This means:

- Two workflows with the same hash are byte-equivalent definitions
- Updating a workflow produces a new hash
- "What ran on this date" is an exact, content-addressable answer
- Change detection between two definitions reduces to hash comparison

Every run records the `definition_hash` of the definition that produced it, so the linkage from a recorded outcome back to the exact definition is unambiguous even after the workflow has been updated.

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

## Reading a definition back

```http
GET /api/workflows/{workflow_name}/definition
```

Returns the stored definition together with its `definition_hash`, `version`, `revision`, and registration metadata.

## Updating a workflow

Definitions are append-only. Registering the same name again with changed content stores a new revision with a new `definition_hash` and makes it the active one. Earlier revisions are never deleted, and previous runs stay associated with the hash they recorded at trigger time, so an update does not affect them retroactively.

Registration keys on content, which gives it three outcomes:

| Outcome | When |
|---|---|
| `unchanged` | The content is already the active revision. Nothing changes and nothing is recorded |
| `versioned` | The content is new for this name. A new revision is created and becomes active |
| `reactivated` | The content matches an earlier revision of this name. That revision becomes active again, with its original `definition_hash` and revision number |

The name is part of the canonical form that the hash is computed over, so the same task graph registered under a different name is a different definition with a different hash, not a copy of the original.

## Running an earlier revision

Re-register that revision's definition under the same name. The hash is a function of the content, so registering the earlier JSON again reproduces its original `definition_hash`, reactivates that revision rather than creating a duplicate, and records the reactivation and the revision it superseded in the audit trail. Triggering by name then runs it.

Reactivation moves the active pointer for that name, so it affects anything else triggering the same workflow. There is no way to run one revision for a single trigger while leaving the active pointer alone, which is the limitation to design around if two callers share a workflow name.

## Triggering a run

```http
POST /api/workflows/{workflow_name}/trigger
Content-Type: application/json

{
  "parameters": {...}
}
```

Returns a run identifier. The run executes asynchronously. See [Runs](runs.md) for status, evidence, and querying.

Triggering is by name, and a run uses the revision currently active under that name. The trigger call takes no `definition_hash`, so a revision cannot be selected for one trigger; to run an earlier one, make it active again as described under [Running an earlier revision](#running-an-earlier-revision). Every run records the `definition_hash` it executed, which is what proves afterwards which revision ran.

## See also

- [Tasks](tasks.md) for the structure of an individual task
- [Runs](runs.md) for what happens when a workflow executes
- [Audit and evidence](audit-and-evidence.md) for what is recorded
