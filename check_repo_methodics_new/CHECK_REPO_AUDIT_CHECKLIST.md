# CHECK_REPO_AUDIT_CHECKLIST

Компактный чек-лист для аудита репозитория. Для каждого пункта ставится `0`, `1`, `2` или `N/A` и фиксируется evidence. `N/A` допустимо только там, где пункт действительно неприменим, например AI/RAG observability для проекта без AI-runtime.

## Scoring

| Score | Значение |
|-------|----------|
| `0` | Нет, противоречиво или вредно устроено. |
| `1` | Есть частично, не покрывает важные зоны или не проверяется. |
| `2` | Есть, используется, поддерживается процессом или автоматическими checks. |

## Section totals

| Section | Items | Max score |
|---------|-------|-----------|
| A. Architecture and structure | 8 | 16 |
| B. Codebase readability and boundaries | 9 | 18 |
| C. Agent instructions and docs | 9 | 18 |
| D. Memory, context and retrieval | 8 | 16 |
| E. Verify and tests | 8 | 16 |
| F. AI/RAG evals and observability | 8 | 16 |
| G. Governance, safety and supply chain | 7 | 14 |
| H. Python 50–100 KLOC specific | 8 | 16 |
| I. Audit output quality | 5 | 10 |
| **Total** | **67** | **134** |

## A. Architecture and structure

| ID | Priority | Проверка | Score | Evidence |
|----|----------|----------|-------|----------|
| A1 | P0 | Есть краткая architecture map: подсистемы, entrypoints, storage, integrations, background jobs. |  |  |
| A2 | P0 | Структура директорий соответствует архитектуре, а не случайной истории проекта. |  |  |
| A3 | P1 | Описаны 3–7 ключевых execution flows with entrypoints, steps, invariants, tests. |  |  |
| A4 | P1 | Есть ADR/decision log для ключевых архитектурных решений. |  |  |
| A5 | P1 | Архитектурные правила имеют статус: enforced/reviewed/aspirational. |  |  |
| A6 | P1 | Бounded contexts and module boundaries visible in code and docs. |  |  |
| A7 | P2 | Есть architecture-as-code / fitness functions для основных boundaries. |  |  |
| A8 | P2 | Есть способ impact analysis: imports/symbol search/dependency graph where needed. |  |  |

## B. Codebase readability and boundaries

| ID | Priority | Проверка | Score | Evidence |
|----|----------|----------|-------|----------|
| B1 | P0 | Доменные модули читаются по дереву директорий без долгой археологии. |  |  |
| B2 | P1 | Public API отделены от internal/impl/adapters. |  |  |
| B3 | P1 | Cross-boundary imports идут через public API или явно запрещены. |  |  |
| B4 | P1 | Side effects локализованы; нет системного import-time IO/global mutable magic. |  |  |
| B5 | P1 | Web/CLI/workers orchestration separated from domain logic. |  |  |
| B6 | P1 | На public boundaries есть honest contracts: types, docstrings, errors, invariants. |  |  |
| B7 | P1 | Modules pass zero-context survival: purpose, API, tests, risks are clear. |  |  |
| B8 | P1 | Нет системной зависимости от god files/generic `utils.py`/catch-all modules. |  |  |
| B9 | P2 | Измеряются file size, complexity, cycles, fan-in/fan-out или хотя бы known hotspots. |  |  |

## C. Agent instructions and docs

| ID | Priority | Проверка | Score | Evidence |
|----|----------|----------|-------|----------|
| C1 | P0 | Root `AGENTS.md` существует и является картой проекта, не энциклопедией. |  |  |
| C2 | P0 | `AGENTS.md` содержит setup, fast/full verify, project map, key docs, forbidden actions. |  |  |
| C3 | P1 | Agent instructions actionable: concrete commands, boundaries, style, output expectations. |  |  |
| C4 | P1 | Есть layered instruction hierarchy без конфликтов: root + local/path-specific/tool-specific. |  |  |
| C5 | P1 | Указан instruction precedence and conflict handling. |  |  |
| C6 | P1 | README для людей and agent docs не конфликтуют and cross-link. |  |  |
| C7 | P1 | Docs use progressive disclosure: map -> local docs -> flows/ADR -> code. |  |  |
| C8 | P2 | Есть single source of truth для setup, verify, architecture, env, migration rules. |  |  |
| C9 | P2 | Есть doc-gardening: broken links, stale docs, TODO/TBD, docs/config consistency. |  |  |

## D. Memory, context and retrieval

| ID | Priority | Проверка | Score | Evidence |
|----|----------|----------|-------|----------|
| D1 | P1 | Durable project memory живёт в Git: ADR, specs, runbooks, flows, eval fixtures where relevant. |  |  |
| D2 | P1 | Repository is system of record or links clearly to external authoritative docs. |  |  |
| D3 | P1 | Нет raw chat logs, session scratchpads, unreviewed summaries, vector dumps, raw embeddings in Git. |  |  |
| D4 | P1 | Context is agent-directed: agent opens relevant docs/files, not automatic context stuffing. |  |  |
| D5 | P1 | Retrieval ladder соответствует размеру/боли проекта: tree/grep/docs/symbols before heavy RAG. |  |  |
| D6 | P2 | Stale/duplicate context is detected and cleaned up. |  |  |
| D7 | P3 | Symbol/dependency graph exists or is intentionally not needed. |  |  |
| D8 | P3 | If repo-RAG exists, corpus policy excludes secrets/noise and requires citations/attribution. |  |  |

## E. Verify and tests

| ID | Priority | Проверка | Score | Evidence |
|----|----------|----------|-------|----------|
| E1 | P0 | Есть single full verify entrypoint. |  |  |
| E2 | P1 | Есть fast local checks: lint, typecheck, unit, targeted tests, smoke. |  |  |
| E3 | P1 | CI and local verify are consistent. |  |  |
| E4 | P1 | Повторяемые процедуры оформлены scripts/tasks, not prose-only. |  |  |
| E5 | P1 | Tests are discoverable and mapped to code zones/flows. |  |  |
| E6 | P2 | Architecture/import checks are part of verify. |  |  |
| E7 | P2 | Failure messages are actionable and link to rules/docs where useful. |  |  |
| E8 | P2 | Runbooks/DEBUG are verified: commands, expected outputs, last checked or owner. |  |  |

## F. AI/RAG evals and observability

Use `N/A` if project does not contain AI-output/RAG/agent runtime. Do not use `N/A` for generic verify/docs/security sections.

| ID | Priority | Проверка | Score | Evidence |
|----|----------|----------|-------|----------|
| F1 | P2 | Для AI/RAG core scenarios есть eval contract: dataset, metrics, thresholds, command. |  |  |
| F2 | P2 | Golden datasets/regression fixtures are versioned and reviewed. |  |  |
| F3 | P2 | Retrieval relevance and generation faithfulness are measured separately. |  |  |
| F4 | P2 | Prompt/config/retriever/corpus versions are tied to evals and traces/logs. |  |  |
| F5 | P2 | Model/tool/retrieval runs have structured logs/traces with run IDs. |  |  |
| F6 | P3 | Error taxonomy separates provider/tool/retrieval/format/content/safety/cost/governance. |  |  |
| F7 | P3 | Content-level observability exists for production AI-output: relevance, faithfulness, safety, format. |  |  |
| F8 | P3 | OTel GenAI mapping/readiness exists or project intentionally stays simpler. |  |  |

## G. Governance, safety and supply chain

| ID | Priority | Проверка | Score | Evidence |
|----|----------|----------|-------|----------|
| G1 | P0 | `.env`/secrets policy documented; `.env.example` exists; no committed secrets. |  |  |
| G2 | P1 | Secret scanning/push protection/pre-commit or equivalent is configured where possible. |  |  |
| G3 | P1 | Required checks/branch protection block merge on critical failures. |  |  |
| G4 | P1 | Critical files/areas have CODEOWNERS or explicit review policy. |  |  |
| G5 | P1 | Dependency policy exists: lockfiles, audit, new dependency rationale. |  |  |
| G6 | P2 | Supply-chain posture is aligned with SSDF/SLSA concepts where relevant. |  |  |
| G7 | P2 | Incidents/post-mortems feed back into tests, docs, rules or eval datasets. |  |  |

## H. Python 50–100 KLOC specific

| ID | Priority | Проверка | Score | Evidence |
|----|----------|----------|-------|----------|
| H1 | P1 | Package layout is consistent (`src/` or flat), imports are predictable. |  |  |
| H2 | P1 | `pyproject.toml` or minimal config set defines build/project/tool settings. |  |  |
| H3 | P1 | Ruff/formatter/linter story is clear and part of verify. |  |  |
| H4 | P1 | Type checker is configured; public interfaces are typed. |  |  |
| H5 | P1 | pytest/tests layout and markers separate unit/integration/slow tests. |  |  |
| H6 | P1 | Config/settings are typed/validated and not read from random call sites. |  |  |
| H7 | P2 | DB migrations/schemas/queue messages/external contracts are documented and tested. |  |  |
| H8 | P2 | Generated code has source-of-truth, regeneration command and edit policy. |  |  |

## I. Audit output quality

| ID | Priority | Проверка | Score | Evidence |
|----|----------|----------|-------|----------|
| I1 | P0 | Audit report includes total score, section scores and readiness interpretation. |  |  |
| I2 | P0 | Every `0`/`1` has concrete evidence with file paths/commands/examples. |  |  |
| I3 | P1 | Recommendations are prioritized P0/P1/P2/P3 with effort/risk/impact. |  |  |
| I4 | P1 | Report separates immediate fixes from advanced/non-goal items. |  |  |
| I5 | P1 | Report does not recommend model/harness/prompt choices outside repo scope. |  |  |

## Score summary template

| Section | Score | Max | Percent | Notes |
|---------|-------|-----|---------|-------|
| A |  | 16 |  |  |
| B |  | 18 |  |  |
| C |  | 18 |  |  |
| D |  | 16 |  |  |
| E |  | 16 |  |  |
| F |  | 16 |  | Use adjusted max if N/A |
| G |  | 14 |  |  |
| H |  | 16 |  |  |
| I |  | 10 |  |  |
| Total |  | 134 |  |  |

## Backlog template

| Priority | Finding IDs | Problem | Evidence | Proposed change | Expected impact | Effort | Risk |
|----------|-------------|---------|----------|-----------------|-----------------|--------|------|
| P0 |  |  |  |  |  |  |  |
| P1 |  |  |  |  |  |  |  |
| P2 |  |  |  |  |  |  |  |
| P3 |  |  |  |  |  |  |  |
