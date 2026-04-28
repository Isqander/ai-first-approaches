# AGENTS.md

> Scope: instructions for AI coding agents working in this repository. Keep this file concise. Put detailed architecture, flows and runbooks in linked docs.

## Project summary

- Purpose: `<what this project does in 2–4 lines>`
- Language/runtime: Python `<version>`
- Main package: `<app or package name>`
- Project type: `<service/library/CLI/data pipeline/AI app/etc.>`

## Read first

1. `README.md` — human overview and quick start.
2. `docs/architecture.md` — system map, boundaries, entrypoints.
3. `docs/flows/` — critical execution flows.
4. `docs/adr/` — architectural decisions.
5. Local `README.md` / `AGENTS.md` in directories you edit.

## Repository map

```text
<path>/                 <purpose>
app/api/                HTTP entrypoints; keep handlers thin
app/domain/             Domain model, policies, use cases
app/infra/              DB, external clients, queues, providers
app/workers/            Background jobs; call domain use cases
tests/                  Unit/integration/architecture tests
docs/                   Architecture, flows, ADRs, runbooks
tools/                  Repo checks and utility scripts
```

## Setup commands

```bash
<install dependencies command>
<optional local env setup command>
```

## Verify commands

Run targeted checks first, then full verify before finalizing changes.

```bash
<fast lint command>
<typecheck command>
<unit tests command>
<targeted tests command>
<architecture/import checks command>
<full verify command>
```

If a command fails, report the failure with the command, relevant output and what you changed. Do not hide failed checks.

## Architecture invariants

- `<domain>` must not import `<infra implementation>` directly.
- HTTP handlers must parse/request-map/call use case/return response; no business rules in handlers.
- Workers and CLI commands call the same use cases as API; do not duplicate domain algorithms.
- External provider SDKs stay in adapters/infra.
- Side effects must not run at import time.
- Cross-domain access goes through public API modules only.

## Coding rules

- Follow existing style and module boundaries.
- Type public interfaces.
- Prefer domain-specific helper modules over generic `utils.py`.
- Keep functions small enough to review; extract cohesive logic instead of adding more branching to god functions.
- Do not edit generated files manually; regenerate from source schema/config.

## Testing rules

- Add/update tests for changed behavior.
- For domain changes, run corresponding unit tests.
- For API/worker changes, run relevant integration or smoke tests if available.
- For architecture boundary changes, run import/architecture checks.

## Documentation rules

- If you change architecture, update `docs/architecture.md` or an ADR.
- If you change a runtime path, update the relevant `docs/flows/*.md`.
- If you change setup/verify/env behavior, update this file, README or configuration docs.
- Avoid duplicating facts. Link to the canonical source instead.

## Dangerous or protected areas

- Secrets and `.env`: never commit real credentials.
- Migrations: require review and tests.
- Security-sensitive code: `<paths>`.
- Generated code: `<paths>`.
- Production config/deploy files: `<paths>`.

## Output expectations

When finishing a task, include:

1. Summary of changes.
2. Files changed.
3. Checks run and results.
4. Known risks or skipped checks.
5. Follow-up items if any.

## Instruction precedence and conflicts

1. Explicit user/maintainer instruction for the current task.
2. This `AGENTS.md`.
3. Local `AGENTS.md` files in directories being edited.
4. Tool-specific instruction files, only when they do not conflict with this file.
5. Older docs only if linked from current docs.

If instructions conflict, stop and report the conflict with file paths. Do not silently choose one.
