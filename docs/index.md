# Validance

Validance is a workflow execution engine for regulated computational environments. It runs containerised tasks defined in JSON or built with the Python SDK, and produces a per-run forensic record suitable as objective evidence under FDA Computer Software Assurance, EudraLex Annex 11, and 21 CFR Part 11.

## What Validance produces

Every workflow run produces, automatically:

- A run record: workflow definition hash, engine version, triggering user, timestamp, parameter values
- For every task: container image and image hash, command, input file content hashes, output file content hashes, exit code, timing, host
- A complete input-step-output lineage retrievable by API
- A tamper-evident audit trail with built-in verification

Records are produced as a side effect of execution. They do not require additional configuration, instrumentation, or manual reconstruction.

## Where to start

If you are evaluating Validance against a regulatory checklist, start with [Regulatory mapping](compliance/regulatory-mapping.md) — what each Validance deliverable corresponds to in Part 11, Annex 11, and FDA CSA terms.

If you are integrating Validance into an existing system, the [Concepts](concepts/workflows.md) section covers workflows, tasks, runs, and audit at the level needed to use the API.

The wire format is documented at [api.validance.io/docs](https://api.validance.io/docs). The Python SDK is at [github.com/validance-io/sdk-python](https://github.com/validance-io/sdk-python) (Apache 2.0).

## Status

Validance is closed-source and pre-GA. The Python SDK is open source under Apache 2.0. The engine is licensed and distributed as a binary; self-hosted and air-gapped deployments are supported.

For pilot enquiries, technical briefings, or evaluation access, contact [wiktor.lisowski@validance.io](mailto:wiktor.lisowski@validance.io).
