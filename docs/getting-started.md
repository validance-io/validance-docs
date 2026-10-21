# Getting started

This page walks you through your first Validance workflow. By the end you will have registered a workflow, uploaded an input file, triggered a run, retrieved the run record, and verified the audit chain.

About fifteen minutes. Uses Validance's hosted evaluation environment at `api.validance.io`, so there is nothing to install.

!!! note "Pre-GA evaluation"
    Validance is pre-GA. The hosted endpoint at `api.validance.io` is open to evaluation traffic without authentication. If you plan to run workloads beyond a quick walkthrough, contact [wiktor.lisowski@validance.io](mailto:wiktor.lisowski@validance.io) to discuss production access.

## What you need

- `curl` and a terminal, or any HTTP client
- About fifteen minutes
- A small text file to use as workflow input

## Setup

```bash
export VALIDANCE_API_URL="https://api.validance.io"
```

Check the endpoint is reachable:

```bash
curl -s "$VALIDANCE_API_URL/api/health"
```

You should get a JSON response with `"status":"healthy"`.

## Step 1: Define a workflow

Validance workflows are JSON. Here is a minimal one with a single task that hashes an input file:

```json
{
  "name": "hello.hashing",
  "version": "1.0",
  "tasks": [
    {
      "name": "hash_file",
      "docker_image": "alpine:3.19",
      "command": "sha256sum inputs/sample/sample.txt > hash.txt",
      "inputs": {},
      "output_files": {
        "hashed": "hash.txt"
      },
      "output_vars": {},
      "depends_on": [],
      "timeout": 60,
      "gate": "auto-approve"
    }
  ]
}
```

Save this as `hello.json`.

What this workflow does:

- Takes one file at trigger time. Validance stages it into the container at `inputs/<role>/<name>` (relative to the task's working directory). The role and name come from the trigger request you'll make in Step 4.
- Runs `sha256sum` on it and writes the result to `hash.txt` in the working directory.
- Declares `hash.txt` as a recorded output (the `output_files` field): the engine collects it after the task completes and records its content hash.

It is small on purpose; the goal is to see the full evidence chain end to end, not to do real work yet.

## Step 2: Register the workflow

```bash
curl -X POST \
     -H "Content-Type: application/json" \
     -d @hello.json \
     "$VALIDANCE_API_URL/api/workflows"
```

The response includes a `definition_hash`: the SHA-256 of the canonical JSON of your workflow. The same JSON always produces the same hash; any change produces a different one. This hash is the workflow's identity, and it appears on every run record produced from this definition.

## Step 3: Upload an input file

```bash
echo "hello world" > sample.txt

curl -X POST \
     -F "file=@sample.txt" \
     "$VALIDANCE_API_URL/api/files/upload"
```

The response includes a `uri` (typically `azure://...`). Note this URI; you will pass it at trigger time. It also includes two identifiers that will reappear on the run record:

- `file_hash` is the SHA-256 of the bytes: proof of which exact content was processed;
- `upload_hash` is the identity of this upload *act*. The upload itself is an audited event, and any run that later stages this file carries it as provenance.

## Step 4: Trigger a run

Substitute the `uri` from Step 3:

```bash
curl -X POST \
     -H "Content-Type: application/json" \
     -d '{
       "input_files": [
         {
           "role": "sample",
           "name": "sample.txt",
           "uri": "azure://...replace-with-uri-from-step-3..."
         }
       ]
     }' \
     "$VALIDANCE_API_URL/api/workflows/hello.hashing/trigger"
```

The response includes a `workflow_hash` (16-char hex), a `session_hash`, and `status: "triggered"` confirming the run was accepted. The `workflow_hash` is the run identifier you will use for status and audit queries.

The run itself progresses through `pending` → `running` → `success` (or `failed`) at the workflow level; individual task states are reported in upper case (`PENDING`, `RUNNING`, `SUCCESS`, `FAILED`) in the same response.

## Step 5: Wait for completion

Poll the run record, addressed by the `workflow_hash` from Step 4:

```bash
curl -s "$VALIDANCE_API_URL/api/runs/<workflow_hash>"
```

(The run hash addresses everything in Validance: this record, the files, the logs, and the audit trail you will query next. If you only know the workflow's name, `GET /api/workflows/hello.hashing/status` returns its latest run.) Once the workflow `status` reaches `success`, you have a completed run record including:

- `workflow_hash`, the run identifier
- `workflow_name`
- `status: "success"` (and per-task `status: "SUCCESS"`)
- `start_time` / `end_time` (UTC, microsecond precision)
- `definition_hash`, bound to this run; identifies the exact workflow definition that produced it
- For each task: `task_name`, `task_hash`, `status`, `docker_image`, `start_time`, `end_time`, `duration_seconds`

The `definition_hash` you saw at registration is bound to this run. Container image, command, and the per-task hash are recorded automatically.

## Step 6: Retrieve the complete run record

One call returns every audit event of the run, merged in chronological order. Substitute your `workflow_hash`:

```bash
curl -s "$VALIDANCE_API_URL/api/audit/<workflow_hash>"
```

The response contains `entity_id`, `entity_type: "workflow"`, `aggregate: true`, `total_events`, and an `events` array: the run's story, compiled automatically at run time as each event happened. For this run the sequence reads:

1. `file.uploaded`: your upload from Step 3, with its content hash in `details.file_hash`
2. `workflow.started`
3. `file.created`: the run log is opened
4. `task.started`: `hash_file`
5. `task.status_changed`: `RUNNING`
6. `file.created`: the task's stdout log
7. `file.created`: your input staged into the container
8. `task.status_changed`: `SUCCESS`
9. `task.completed`: the full provenance record (shown below)
10. `file.created`: the output `hash.txt`
11. `variable.set`: the output registered as a workflow variable
12. `workflow.status_changed`: `SUCCESS`
13. `file.created`: the final run log

Each event is a complete record. Here is a real `task.completed` (because the walkthrough input is the exact bytes `hello world\n`, the two content hashes below are the same ones you will see):

```json
{
  "id": 273150,
  "event_type": "task.completed",
  "entity_type": "task",
  "entity_id": "cf8ab65b6086452a",
  "actor": "system",
  "timestamp": "2026-10-21T21:10:38.901553",
  "details": {
    "command": "sha256sum inputs/sample/sample.txt > hash.txt",
    "docker_image": "alpine:3.19",
    "duration_seconds": 0.289,
    "exit_code": 0,
    "input_file_hashes": {
      "inputs/sample/sample.txt": "a948904f2f0f479b8f8197694b30184b0d2ed1c1cd2a1ec0fb85d299a192a447"
    },
    "output_file_hashes": {
      "hash.txt": "4c24f90c0471d30dab8c2a66d677a6b32f094f84388eb3e289c31a2a07ff3d75"
    },
    "status": "success",
    "task_name": "hash_file",
    "workflow_hash": "<your workflow_hash>"
  },
  "event_hash": "a6eef49a58180c84566dded3f7de33a1...",
  "previous_event_hash": "884e5e63b53a9dcfcaab062f975a5b85...",
  "previous_entity_hash": "884e5e63b53a9dcfcaab062f975a5b85..."
}
```

Every event carries its actor, a UTC microsecond-precision timestamp, its own hash, and pointers to its predecessors. Every event is recorded against the specific thing it happened to (the run, the task, one particular use of a file, your upload), so the trail contains exactly this run's history. The event count varies slightly with log-file handling; what matters is that the upload, every state transition, every file, and the full task provenance are all present, in order, from one call.

Anchoring the same endpoint on a task hash returns just that task's slice. Anchoring it on a file's *content hash* (the 64-character SHA-256) answers a different question: every upload and every use of those exact bytes, across all runs. That is the lineage view.

## Step 7: Verify the chains

Every event is hash-chained to its predecessors, in a global chain (system-wide) and a per-entity chain (this run, this task, this file use). Verification recomputes the hashes and walks the chains:

```bash
curl -s "$VALIDANCE_API_URL/api/audit/<workflow_hash>/verify"
```

Response (abridged; one record per entity in the run, including each file use):

```json
{
  "entity_id": "<workflow_hash>",
  "entity_type": "workflow",
  "aggregate": true,
  "all_intact": true,
  "entities": [
    {"entity_type": "workflow", "entity_id": "<workflow_hash>", "valid": true, "total_events": 2, "break_at": null, "details": "All hashes verified"},
    {"entity_type": "task",     "entity_id": "<task_hash>",     "valid": true, "total_events": 4, "break_at": null, "details": "All hashes verified"},
    {"entity_type": "file",     "entity_id": "...",             "valid": true, "total_events": 1, "break_at": null, "details": "All hashes verified"}
  ]
}
```

`all_intact: true` is the one-line answer to "has anything in this run's record been altered since it was written?" Each entity's chain is verified independently; if a record had been modified, that entity would report `valid: false` with `break_at` pointing at the broken link. Only that entity turns invalid, so you'd know exactly what was touched.

For the system-wide check (every event the engine has ever recorded, one chain), use the anchor-less form:

```bash
curl -s "$VALIDANCE_API_URL/api/audit/verify"
```

## What you have just demonstrated

In the records above, all captured automatically:

| Captured | Where you saw it |
|---|---|
| Exact workflow definition | `definition_hash`: registration response (Step 2), status (Step 5) |
| Exact run instance | `workflow_hash`: trigger response (Step 4), status (Step 5), audit (Step 6) |
| Container image used | `docker_image`: status (Step 5), `task.completed` details (Step 6) |
| Exact input file by content | `file_hash`: upload response (Step 3), `file.uploaded` / `file.created` / `task.completed` details (Step 6) |
| Where the input came from | `upload_hash`: upload response (Step 3); the upload event appears in the run record (Step 6) |
| Exact output file by content | output `file_hash`: `file.created` / `task.completed` details (Step 6), file list (`GET /api/files/<workflow_hash>`) |
| Exit code per task | `exit_code`: `task.completed` details (Step 6) |
| Time of every state change | `start_time` / `end_time` per task and per run; event timestamps in the trail |
| Complete run trail, one call | `GET /api/audit/<workflow_hash>` (Step 6): merged, chronological, no client-side assembly |
| Tamper-evident record | `verify` endpoints (Step 7): per-run `all_intact` and global chain verification |

You produced this in seven HTTP calls without any QA-side instrumentation. It is the substrate the [Regulatory mapping](compliance/regulatory-mapping.md) page describes in clause-by-clause terms.

## Where to go next

- [Workflows](concepts/workflows.md) for more on workflow definition and registration
- [Tasks](concepts/tasks.md) for task structure including dependencies, approvals, secrets, and resource constraints
- [Audit and evidence](concepts/audit-and-evidence.md) for full audit and lineage retrieval
- [Regulatory mapping](compliance/regulatory-mapping.md) for how the artefacts you just produced map to specific regulatory clauses
- The [Python SDK](https://github.com/validance-io/sdk-python) if you would rather construct workflows in Python than write JSON directly
