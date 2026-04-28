# 05. Python 50–100 KLOC Specific

Адаптация общей методики под Python-проект среднего размера. Цель — не навязать конкретный framework, а проверить, что кодовая база предсказуема для AI-агента: package layout, imports, typing, tests, tooling, entrypoints, configuration and side effects.

## Карта методик

| ID | Методика | Главный вопрос |
|----|----------|----------------|
| P1 | Package layout | Есть ли единая схема пакетов и импортов? |
| P2 | `pyproject.toml` and tool config | Инструменты собраны предсказуемо? |
| P3 | Type surface | Типизированы ли публичные границы? |
| P4 | Import boundaries | Контролируются ли forbidden imports, cycles, layer violations? |
| P5 | Entrypoint separation | Web/CLI/workers/domain/infra разведены? |
| P6 | Config and environment | `.env`, settings and secrets policy ясны? |
| P7 | Tests and fixtures | Тесты структурированы и запускаются локально? |
| P8 | Data migrations and schemas | Миграции/схемы/очереди отражены в docs and verify? |
| P9 | Dependency policy | Dependencies pinned, reviewed, audited? |
| P10 | Module hygiene | Нет god files, magic globals, catch-all utils? |
| P11 | Generated code | Generated artifacts имеют правила и owners? |
| P12 | AI/RAG Python code | Если есть AI-runtime, он изолирован, наблюдаем и тестируем? |

## P1. Package layout

**Что проверяем.** Есть понятная структура пакетов. Она может быть `src/` layout или flat layout, но должна быть consistent.

Пример для service/app проекта:

```text
app/
  api/            # HTTP handlers/controllers, thin layer
  cli/            # CLI commands
  workers/        # background jobs
  domain/         # domain model, use cases, policies
  infra/          # DB, external clients, queues, model providers
  settings/       # config parsing and validation
  shared/         # truly shared low-level primitives only
  __init__.py
tests/
  unit/
  integration/
  architecture/
docs/
  architecture.md
  flows/
  adr/
```

Пример для library проекта:

```text
src/package_name/
  public_module.py
  subdomain_a/
  subdomain_b/
  _internal/
tests/
  unit/
  integration/
```

**Red flags.**

- `app.py` / `main.py` на 1000+ строк.
- `utils.py`, `helpers.py`, `common.py` как центральные узлы проекта.
- Смешение `api`, `domain`, `infra` в одном файле.
- Пакеты названы технически (`services`, `managers`) без доменных boundaries.

## P2. `pyproject.toml` and tool config

Python Packaging User Guide описывает `pyproject.toml` как конфигурационный файл для packaging tools, build-system, project metadata и tool-specific tables [E14]. Для 50–100 KLOC это полезная точка консолидации.

**Что проверяем.**

- Есть `[build-system]`.
- Есть `[project]` или понятный альтернативный metadata source.
- Tool configs собраны в `pyproject.toml` или в минимальном наборе файлов.
- Команды verify ссылаются на эти configs.
- Нет расхождений между CI and local tools.

**Пример минимального набора.**

```toml
[tool.ruff]
line-length = 100
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP", "SIM"]

[tool.mypy]
python_version = "3.11"
warn_unused_ignores = true
no_implicit_optional = true

[tool.pytest.ini_options]
addopts = "-ra"
testpaths = ["tests"]
markers = [
  "integration: tests that require external services",
  "slow: long-running tests",
]
```

Ruff provides a fast linter/formatter with `ruff format` as formatter entrypoint [E15]. Mypy supports gradual typing and can find type errors without running code [E16]. Конкретные инструменты можно заменить, но **должен быть один obvious tooling story**.

## P3. Type surface

**Что проверяем.** Публичные интерфейсы типизированы, даже если внутри проекта gradual typing.

**Обязательный минимум.**

- Public functions/classes in domain/services.
- Ports/interfaces/protocols.
- DTOs and config objects.
- External adapter methods.
- AI/RAG result schemas, if present.
- Exceptions and error result types where relevant.

**Не обязательно сразу.** Полная strict typing всего legacy-кода. Для 50–100 KLOC часто лучше расширять typed surface по boundaries и hotspots.

**Red flags.**

- Public APIs принимают/возвращают `dict[str, Any]` без schema.
- `Any` используется, чтобы скрыть проблемы на границах.
- Type checker настроен, но исключает core packages.
- Type errors игнорируются массово без owner/plan.

## P4. Import boundaries

**Что проверяем.** Импортная структура соответствует архитектуре.

**Минимальные checks.**

- Domain не импортирует infra implementations.
- Web/API не импортирует worker internals.
- Cross-domain imports идут через public API.
- Нет import cycles.
- `internal/_private/impl` не импортируется извне.

Import Linter позволяет imposing constraints on imports between Python modules [E17], а layers contract enforces layered architecture and indirect imports count [E18]. Это не единственный вариант, но хороший practical baseline.

**Пример pytest custom check, если import-linter не подходит.**

```python
# tests/architecture/test_no_private_imports.py
from pathlib import Path

ROOT = Path(__file__).resolve().parents[2]


def test_no_external_private_imports():
    offenders = []
    for path in (ROOT / "app").rglob("*.py"):
        text = path.read_text()
        if "from app.orders._" in text and "app/orders" not in str(path):
            offenders.append(str(path.relative_to(ROOT)))
    assert not offenders, "Forbidden private imports from app.orders: " + ", ".join(offenders)
```

Better to use AST/import graph for real checks, но даже простой check лучше prose-only rule.

## P5. Entrypoint separation

**Что проверяем.** Разные entrypoints вызывают общие use cases, а не дублируют domain logic.

| Entrypoint | Должен делать | Не должен делать |
|------------|---------------|------------------|
| HTTP handler | Parse/auth/validate request, call use case, map response | SQL, provider SDK calls, business rules |
| CLI command | Parse args, call use case, print result | Дублировать API logic |
| Worker/job | Deserialize job, call use case, manage retries | Менять domain state напрямую |
| Scheduled task | Orchestrate batch use case | Содержать всю batch business logic inline |
| AI/RAG endpoint | Validate input, call AI service, enforce policy | Строить prompt/retrieval ad hoc in handler |

**Red flags.**

- Три entrypoints реализуют один алгоритм по-разному.
- Handler directly imports database session and external clients.
- Worker and API share code by importing each other's internals.

## P6. Config and environment

**Что проверяем.** Конфигурация очевидна, typed/validated, secrets не попадают в Git.

**Минимум.**

- `.env.example` without real secrets.
- `docs/configuration.md` or section in README.
- Settings object/module with validation.
- Clear policy: local/dev/staging/prod.
- No hidden import-time config side effects.
- Secret scanning/push protection if available.

**Red flags.**

- `.env` committed.
- Settings are read from environment at random call sites.
- Agent cannot run tests because required env vars are undocumented.
- Config docs and code disagree.

## P7. Tests and fixtures

**Что проверяем.** Тесты позволяют быстро проверить локальное изменение.

**Good signals.**

- `tests/unit/<domain>` mirrors source domains.
- `tests/integration` separated and marked.
- Fixtures named by domain, not generic `data1`.
- Mocks/fakes live in predictable location.
- Test commands documented in `AGENTS.md`.
- Test failures include readable assertion messages.

**Minimum commands.**

```bash
just test-unit
just test-integration
just test-path app/orders
just test-smoke
```

**Red flags.**

- Tests rely on order or shared mutable state.
- Fixtures are huge snapshots without explanation.
- Integration tests run by default and fail without external services.
- Agent cannot identify which tests cover changed code.

## P8. Data migrations, schemas and queues

**Что проверяем.** Stateful parts are documented and tested:

- DB migrations and migration policy.
- Pydantic/dataclass/schema contracts.
- Queue topics/messages.
- Background job retries/idempotency.
- External API contract fixtures.
- Backward compatibility rules.

**Evidence.**

- `docs/flows/*.md` mention migrations/messages where relevant.
- Migration commands are executable.
- There are tests for schema compatibility and idempotency.
- Dangerous migrations require review/CODEOWNERS.

**Red flags.**

- Agent changes model but not migration.
- Queue payload schema lives only in tribal knowledge.
- Callback/idempotency behavior undocumented.

## P9. Dependency policy

**Что проверяем.** Dependencies are controlled.

**Minimum.**

- Lockfile or reproducible dependency management.
- New dependencies require rationale.
- Dependency audit command.
- Clear dev/prod extras separation.
- Vendor SDKs isolated in infra adapters.
- SBOM/provenance if shipped externally or required.

SLSA and SSDF provide vocabulary/checklists for supply-chain integrity and secure development practices [E19][E20][E21]. For this methodic, the practical check is: **can the agent add/update a dependency without bypassing review, audit and lockfile policy?**

## P10. Module hygiene

**God files.** Files `>500` lines are not automatically bad, but must be justified: generated code, large schema, table of constants, parser, etc. In core domain, a 500-line file usually deserves decomposition.

**Generic helpers.**

- `utils.py` is acceptable only for truly generic, low-level helpers.
- Domain helpers should live in domain-named modules.
- IO helpers should live in infra/adapters.
- Test helpers should not leak into production packages.

**Import-time behavior.**

- Avoid DB/network calls at import.
- Avoid mutable global registries without lifecycle.
- Avoid hidden monkeypatching.

**Agent check.** Найти top 20 largest files, top 20 most imported modules, modules with `Any`, `globals`, broad `except`, and generic names.

## P11. Generated code

**Что проверяем.** Generated artifacts have rules.

**Questions.**

- Где source of truth: schema, OpenAPI, protobuf, SQL, prompt template?
- Как пересобрать generated code?
- Можно ли редактировать generated files вручную?
- Есть ли marker/header?
- Есть ли tests against generated contracts?

**Red flags.**

- Agent edits generated file manually.
- Generated code mixed with handwritten logic.
- Нет команды regeneration.

## P12. AI/RAG Python code

Если внутри Python-проекта есть AI/RAG logic, она должна быть изолирована и проверяема.

**Good shape.**

```text
app/ai/
  prompts/             # versioned templates/configs, if part of product
  retrieval/           # retrievers, chunking, corpus config
  generation/          # response generation, schemas
  evals/               # eval runners or links to evals/
  observability.py     # trace/log helpers
```

Или это может быть `app/infra/ai`, `app/rag`, `app/llm`. Название неважно; важна граница.

**Rules.**

- Prompt/config version visible in logs/evals.
- Retrieval corpus policy documented.
- RAG answers cite context where relevant.
- Tool/model calls are isolated, mockable, and traced.
- Output validation exists before downstream use.
- Eval command exists for core scenarios.
- Safety policy covers prompt injection, secrets, tool permissions.

OWASP LLM risks include prompt injection, insecure output handling, sensitive information disclosure and insecure plugins [E10]. Для Python-кода это означает: validate outputs, restrict tools, redact logs, avoid passing raw model output into dangerous operations.

## Итоговые вопросы для агента

1. Понятна ли структура пакетов без чтения всего проекта?
2. Собраны ли tool configs and verify commands в предсказуемом месте?
3. Типизированы ли public boundaries?
4. Контролируются ли imports/layers/cycles?
5. Разделены ли entrypoints, domain logic and infra?
6. Понятны ли config/env/secrets rules?
7. Тесты локализуют изменения?
8. Есть ли правила для migrations, schemas, generated code?
9. Если есть AI/RAG logic, есть ли evals, tracing and safety boundaries?
