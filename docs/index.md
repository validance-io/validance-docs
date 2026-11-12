# Validance

Validance is a workflow execution engine for regulated computational environments. It runs containerised tasks defined in JSON or built with the Python SDK, and produces a per-run forensic record suitable as objective evidence under FDA Computer Software Assurance, EudraLex Annex 11, and 21 CFR Part 11.

## What Validance produces

Every workflow run produces, automatically:

- A run record: run identifier, workflow name, workflow definition hash, definition version, status, parameter values, start and end timestamps
- Per task: the command that ran, the container image and the `sha256` digest of the image manifest that actually executed, the exit code, the duration, and a content hash for every input and output file
- The digest of the engine image that performed the run, recorded as bytes rather than as a version string
- For every file the run touched: a SHA-256 content hash, its size, the task it belongs to, and whether it was an input or an output
- A lineage view for any file, keyed by its content hash: where those bytes entered the system, and every run and task that consumed them
- Captured standard output and standard error per task, and the output variables each task wrote
- A tamper-evident audit trail, hash-chained both globally and per entity, with verification exposed as an API call

Records are produced as a side effect of execution. They do not require additional configuration, instrumentation, or manual reconstruction.

## Where to start

If you are evaluating Validance against a regulatory checklist, start with [Regulatory mapping](compliance/regulatory-mapping.md), which sets out what each Validance deliverable corresponds to in Part 11, Annex 11, and FDA CSA terms, and what it does not.

If you want to run something, [Getting started](getting-started.md) walks a first workflow end to end against a live engine.

If you are integrating Validance into an existing system, the [Concepts](concepts/workflows.md) section covers workflows, tasks, runs, and audit at the level needed to use the API.

The wire format is documented at [api.validance.io/docs](https://api.validance.io/docs). The Python SDK is at [github.com/validance-io/sdk-python](https://github.com/validance-io/sdk-python) (Apache 2.0).

## Status

Validance is closed-source and pre-GA. The Python SDK is open source under Apache 2.0. The engine is licensed and distributed as a binary; self-hosted and air-gapped deployments are supported.

For pilot enquiries, technical briefings, or evaluation access, contact [wiktor.lisowski@validance.io](mailto:wiktor.lisowski@validance.io).
