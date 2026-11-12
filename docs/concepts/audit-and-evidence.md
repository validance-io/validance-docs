# Audit and evidence

Validance captures a structured audit trail of every workflow run. Audit records are persistent, queryable, and tamper-evident.

## What is captured

For every workflow run, the audit trail contains time-stamped records of:

- Workflow lifecycle: `workflow.started`, `workflow.status_changed`
- Task lifecycle: `task.started`, `task.status_changed`, `task.completed`
- Files: `file.uploaded`, `file.created`, each carrying the file's SHA-256 content hash in its details
- Variables: `variable.set`
- Approvals: the request and its resolution, recorded against an `approval` entity

Completion is recorded as a status change rather than as a separate completion event, so a finished run ends with `workflow.status_changed`.

Each audit event includes:

- An event identifier and the event type
- The entity the event refers to, as an `entity_type` and `entity_id` pair. Entity types in use are `workflow`, `task`, `file` and `approval`
- The actor. This is a system identity, not a named operator: see [Operator identity](../compliance/regulatory-mapping.md#operator-identity)
- A timestamp with microsecond precision, in a fixed format
- Event-specific details, as structured JSON
- `event_hash`, `previous_event_hash` and `previous_entity_hash`, which are the chain pointers described below

## Tamper-evident records

Every event is hash-chained twice. `previous_event_hash` links it to the preceding event across the whole log, and `previous_entity_hash` links it to the preceding event for the same entity. The first event on either chain carries `genesis`.

Two chains rather than one means a single workflow's or a single task's history can be verified on its own, without walking or trusting the rest of the log. Modifying any record after it has been written breaks the linkage, and the break is detectable without privileged access.

## Retrieving evidence for a run

```http
GET /api/audit/{workflow_hash}
```

Returns the audit trail for that run in chronological order, including its task, file, variable and approval events, with a `total_events` count. The response is structured JSON suitable for direct inclusion in QA review packages.

The same endpoint serves any entity, not only a run. Pass `entity_type` to disambiguate an identifier, and `aggregate` to choose how much is folded in.

Aggregation is on by default: a run's own events are returned together with the events of the tasks and files belonging to it. Passing `aggregate=false` returns only the events recorded against the anchor entity itself, which for a workflow is its start and its status change. For one run in the example above that is the difference between thirteen events and two.

## Verifying the audit trail

The global chain, with no anchor entity:

```http
GET /api/audit/verify
```

```json
{
  "valid": true,
  "total_events": 100,
  "break_at": null,
  "details": "All hashes verified"
}
```

A single entity and the entities belonging to it:

```http
GET /api/audit/{entity_id}/verify
```

```json
{
  "entity_id": "50a4b5f00150139f",
  "entity_type": "workflow",
  "aggregate": true,
  "all_intact": true,
  "entities": [
    {
      "entity_type": "workflow",
      "entity_id": "50a4b5f00150139f",
      "valid": true,
      "total_events": 2,
      "break_at": null,
      "details": "All hashes verified"
    },
    {
      "entity_type": "task",
      "entity_id": "89195bfec7559e6b",
      "valid": true,
      "total_events": 4,
      "break_at": null,
      "details": "All hashes verified"
    }
  ]
}
```

The per-entity form verifies each chain separately and reports `all_intact` across them, so a break is localised to the entity whose chain failed. If a break is found, `valid` is false for that entity and `break_at` carries the index at which the chain stopped matching.

The entity type is inferred from the shape of the identifier when `entity_type` is omitted. There is no silent fall back to the global chain: if the identifier cannot be resolved to an entity, the request fails rather than quietly verifying something else.

## Files and their hashes

Content hashes are the anchor that ties a recorded outcome to the bytes that produced it. They are recorded in the details of file events, and are also listed directly for a run:

```http
GET /api/files/{workflow_hash}
```

Each entry carries the file name, its SHA-256 `file_hash`, its size, the task it belongs to, and `is_input` / `is_output` flags. Reading the input and output hashes per task is how you see which bytes entered a step and which left it.

## Lineage

Lineage is answered by the audit endpoint, anchored on a file's content hash rather than on a run:

```http
GET /api/audit/{content_hash}
```

A 64-character hex identifier is read as a content anchor, and the response is a lineage view over those exact bytes. A content hash is cross-run by nature, so the answer is not confined to one execution:

```json
{
  "entity_type": "content",
  "uploads": [
    { "upload_hash": "28c946a723d51ee5", "file_name": "sample.txt", "uri": "..." }
  ],
  "uses": [
    { "file_use_hash": "8a962778012134ec",
      "workflow_hash": "50a4b5f00150139f",
      "task_hash": "89195bfec7559e6b",
      "file_name": "inputs/sample/sample.txt",
      "source_upload_hash": "28c946a723d51ee5" }
  ],
  "events": [ "..." ]
}
```

`uploads` is where those bytes entered the system. `uses` is every place they were consumed, one entry per use, each carrying the run and the task that used it and a `source_upload_hash` pointing back to the upload it came from. `events` holds the audit records for those acts.

The view is use-centric rather than run-centric, so the question it answers directly is where a file came from and everywhere it went. For a whole-run picture, combine the run's file list, which gives the hashes with their task and their input or output role, with the content-anchored view for the hashes that matter.

## Retention

Audit records are persistent. Validance does not provide an interface for deleting individual audit events; records are retained for the lifetime of the database. Retention beyond the engine's lifetime is an operational concern handled at the database backup and archival level.

## Inseparability of audit and execution

Audit emission is part of the run's execution path. If an audit event cannot be persisted, the run does not proceed. For any successfully completed run the corresponding audit record is present; for any failed run the failure is recorded.

There is no operating mode in which a workflow runs but its audit trail is silently absent.

## Use in regulatory contexts

For how the audit trail and run records map to specific regulatory clauses, and for the clauses they do not satisfy, see [Regulatory mapping](../compliance/regulatory-mapping.md).

## See also

- [Runs](runs.md) for what is captured per run
- [Regulatory mapping](../compliance/regulatory-mapping.md) for the regulatory mapping
