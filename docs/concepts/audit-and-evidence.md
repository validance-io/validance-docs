# Audit and evidence

Validance captures a complete, structured audit trail of every workflow run. Audit records are persistent, queryable, and tamper-evident.

## What is captured

For every workflow run, the audit trail contains time-stamped, attributable records of:

- Workflow lifecycle events: started, status changes, completed
- Task lifecycle events: started, status changes, completed
- File events: file creation, content hashes recorded
- Variable events: variables written during execution
- Approval events: approval gates triggered and resolved

Each audit event includes:

- Event type
- The entity (workflow or task) the event refers to
- The actor (the user or system that caused the event)
- Timestamp in UTC, microsecond precision, fixed format
- Event-specific details

## Tamper-evident records

Audit records are linked cryptographically. Modifying any record after it has been written breaks the linkage, and the break is detectable without privileged access.

## Verifying the audit trail

```http
GET /api/audit/verify
GET /api/audit/verify?entity_type=workflow&entity_id={hash}
GET /api/audit/verify?entity_type=task&entity_id={hash}
```

Returns:

```json
{
  "valid": true,
  "total_events": 1247,
  "break_at": null,
  "details": "All records verified"
}
```

If a break is found, the response includes its index and a description.

Verification is available for the entire system, for a specific workflow's history, or for a single task's history.

## Retrieving evidence for a specific run

The most common evidence query is for a single run:

```http
GET /api/runs/{run_id}/audit
```

Returns the full audit trail for that run, including all task events, file events, variable events, and any approval events, in chronological order. The response is structured JSON suitable for direct inclusion in QA review packages.

Lineage as a graph:

```http
GET /api/runs/{run_id}/lineage
```

Returns the input-step-output graph: which files entered the workflow, which task consumed them, what each task produced, which files left.

## Retention

Audit records are persistent. Validance does not provide an interface for deleting individual audit events; records are retained for the lifetime of the database. Retention beyond the engine's lifetime is an operational concern handled at the database backup and archival level.

## Inseparability of audit and execution

Audit emission is part of the run's execution path. If an audit event cannot be persisted, the run does not proceed. For any successfully completed run the corresponding audit record is present; for any failed run the failure is recorded.

There is no operating mode in which a workflow runs but its audit trail is silently absent.

## Use in regulatory contexts

For how the audit trail and run records map to specific regulatory clauses, see [Regulatory mapping](../compliance/regulatory-mapping.md).

## See also

- [Runs](runs.md) for what is captured per run
- [Regulatory mapping](../compliance/regulatory-mapping.md) for the regulatory mapping
