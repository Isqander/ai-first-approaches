# CHECK_REPO_METHODICS

Методика проверки Python-репозитория размером примерно **50–100 KLOC** на пригодность к работе AI-агентов разработки. Документ оценивает **сам проект как среду**, в которой агент читает, понимает, меняет и проверяет код.

Методика не выбирает модель, провайдера, агентный harness, IDE, MCP-серверы или рабочие промпты для реализации фич. Это намеренно вынесено за рамки: такие решения меняются быстрее, чем структура проекта, а задача аудита — сделать репозиторий понятным и проверяемым для разных агентов.

## Что получилось в этом пакете

| Файл | Назначение |
|------|------------|
| `CHECK_REPO_METHODICS.md` | Карта пакета, правила оценки, минимальный набор, порядок работы агента. |
| `CHECK_REPO_AUDIT_CHECKLIST.md` | Табличный чек-лист: score `0/1/2`, priority, evidence. |
| `methodics/01_architecture_and_code.md` | Архитектура, structural alignment, codebase topology, deep modules, boundaries. |
| `methodics/02_agents_md_and_docs.md` | `AGENTS.md`, layered instructions, docs as interface, source of truth. |
| `methodics/03_memory_rag_context.md` | Что считать памятью репозитория, где нужен/не нужен RAG, retrieval ladder. |
| `methodics/04_verify_observability_governance.md` | Verify loop, evals, architecture fitness functions, observability, safety/governance. |
| `methodics/05_python_50_100_kloc.md` | Python-specific адаптация: package layout, `pyproject.toml`, typing, imports, tests. |
| `methodics/06_agent_audit_protocol.md` | Пошаговый протокол аудита для AI-агента и формат итогового отчёта. |
| `templates/AGENTS.template.md` | Шаблон хорошего `AGENTS.md` для репозитория. |
| `templates/CHECK_REPO_REPORT.template.md` | Шаблон отчёта по итогам аудита. |
| `SOURCES.md` | Внешние и архивные источники, использованные при сборке методики. |

## Как пользоваться агенту

1. Открой этот файл и `CHECK_REPO_AUDIT_CHECKLIST.md`.
2. Прочитай `methodics/06_agent_audit_protocol.md` и собери evidence: дерево проекта, docs, `AGENTS.md`, `pyproject.toml`, verify-команды, тесты, import rules, ключевые execution flows.
3. Оцени каждый пункт чек-листа по шкале `0/1/2`.
4. Для проваленных блоков прочитай соответствующий тематический файл из `methodics/`.
5. Сформируй отчёт по шаблону `templates/CHECK_REPO_REPORT.template.md`.
6. Предлагай изменения в порядке: **instructions/docs → verify → architecture constraints → code structure → advanced retrieval/observability**.

Главное правило: **каждая оценка должна иметь evidence**. Нельзя писать «архитектура неясная» без примера путей, импортов, конфликтующих документов или проваленных checks.

## Шкала оценки

Для каждого пункта чек-листа ставится:

| Score | Значение |
|-------|----------|
| `0` | Отсутствует, противоречиво, вредно устроено или агент не может этим пользоваться. |
| `1` | Есть частично, не покрывает важные зоны, не проверяется автоматически или быстро устаревает. |
| `2` | Есть, используется, поддерживается процессом или автоматическими checks, даёт агенту actionable guidance. |

Интерпретация итогового результата:

| Доля от максимума | Интерпретация |
|-------------------|---------------|
| `85%+` | Strong AI-friendly repository: агент может работать системно, а не только локально. |
| `65–84%` | Usable: работать можно, но есть заметные расходы на восстановление контекста. |
| `45–64%` | Local-only AI usefulness: агент помогает на отдельных задачах, но системно спотыкается. |
| `<45%` | Сначала чинить структуру, документацию и verify-loop; продвинутые RAG/агенты пока преждевременны. |

## Минимальный обязательный набор для 50–100 KLOC

Если нужно быстро подготовить репозиторий к агентной разработке, сначала должны появиться эти артефакты:

1. **Карта архитектуры на 1–2 страницы**: подсистемы, entrypoints, storage, внешние интеграции, фоновые задачи.
2. **3–7 execution flows**: как запрос/команда/джоба проходит через систему.
3. **`AGENTS.md` как навигация**, а не энциклопедия: setup, verify, структура, ключевые docs, запреты, стиль.
4. **Предсказуемый package layout**: структура директорий отражает домены и слои, а не историю случайных изменений.
5. **Явные public API и boundaries**: куда агенту можно импортировать, а куда нельзя.
6. **Единая команда verify**: например `make verify`, `just verify`, `tox`, `nox`, task runner или скрипт.
7. **Быстрый локальный loop**: отдельные `lint`, `typecheck`, `test:unit`, `test:smoke`.
8. **Минимальные architecture tests / fitness functions**: import rules, layer rules, cycle checks, threshold checks.
9. **Doc-gardening**: проверка битых ссылок, stale docs, TODO/TBD, противоречивых duplicated rules.
10. **Security baseline**: `.env` policy, secrets scanning/push protection, dependency scanning, protected branches.

## Основные принципы методики

### 1. Репозиторий — это не только код, но и рабочая память агента

Агент не должен восстанавливать проект из случайной смеси кода, README, старых чатов и устных договорённостей. Устойчивые знания должны жить в Git: ADR, specs, runbooks, post-mortems, eval fixtures, architecture rules. При этом Git не должен становиться свалкой raw chat logs, embeddings и session scratchpads. См. `methodics/03_memory_rag_context.md`.

### 2. `AGENTS.md` — карта, не свалка инструкций

Современная практика сходится на том, что agent instruction files должны давать setup/test/style/conventions и project-specific context, но не превращаться в книгу обо всём проекте [E1][E2][E3]. Для крупных проектов нужен layered подход: root `AGENTS.md`, локальные инструкции по директориям, отдельные docs для деталей.

### 3. Архитектура должна совпадать с кодом

Если docs говорят «domain/service/infrastructure», а дерево проекта говорит `utils/`, `common/`, `misc/`, агент будет ошибаться. Проверка идёт по цепочке: **docs → directory map → imports → tests → runtime traces**. См. `methodics/01_architecture_and_code.md`.

### 4. Архитектурные правила должны исполняться

Markdown-правило «не импортировать infra из domain» не удержит агента. Нужны import rules, architecture tests, custom pytest checks, CI gates или другие fitness functions. Architecture-as-code и fitness functions дают objective feedback по architecture intent [E17][E18][E25][E26].

### 5. Long context и RAG не заменяют структуру

Для 50–100 KLOC обычно нужна не «векторная магия», а нормальная лестница retrieval: file tree → grep/symbol search → local docs → hybrid search → code graph. RAG нужен тогда, когда есть измеримый retrieval pain и eval-контур. См. `methodics/03_memory_rag_context.md`.

### 6. Verify loop важнее красивых инструкций

Агент должен уметь проверить изменение без угадывания ритуалов. Единая команда verify, быстрые partial checks, actionable errors и architecture checks уменьшают число ложных правок.

### 7. Если проект сам содержит AI/RAG-логику, качество нужно измерять отдельно

Для AI-output мало обычного uptime. Нужны eval datasets, regression cases, retrieval relevance, answer faithfulness, format validity, safety checks и traceability prompt/config versions. Метрики faithfulness/relevance отдельно проверяют генерацию поверх retrieved context и качество retrieval [E11][E12][E13].

### 8. Safety и governance должны быть enforced, а не только описаны

Критические ограничения должны быть в CI, hooks, scanners, permissions, branch protection, architecture rules или runtime policy. Для supply-chain/security baseline полезны SSDF, SLSA, secret scanning/push protection и OWASP LLM risk taxonomy [E10][E19][E20][E21][E22].

## Priority model

| Priority | Что означает | Примеры |
|----------|--------------|---------|
| `P0` | Без этого агент часто будет ломать проект или не сможет безопасно работать. | Нет `AGENTS.md`, нет verify, хаотичная структура, нет `.env` policy. |
| `P1` | Базовая инженерная пригодность для стабильной агентной разработки. | Architecture map, public API boundaries, typing on interfaces, local README. |
| `P2` | Усиление качества и скорости итераций. | Architecture tests, doc-gardening, evals, structured logs. |
| `P3` | Зрелая AI-first инфраструктура. | Symbol graph, hybrid retrieval, OTel GenAI conventions, content-level observability. |

## Что не включать в аудит

Не оценивать и не рекомендовать в рамках этой методики:

- какую LLM/нейросеть использовать;
- какой agent harness/IDE/platform выбрать;
- какие конкретные рабочие промпты писать для реализации фич;
- какие MCP-серверы подключать;
- как организовать multi-agent workflow вне репозитория;
- какой коммерческий observability/eval продукт купить.

Можно оценивать только то, что связано с **репозиторием как инженерной средой**: docs, instructions, layout, verify, tests, boundaries, memory artifacts, RAG/eval/observability when present, security/governance gates.

## Быстрый порядок исправлений после аудита

1. **P0 instructions + verify**: root `AGENTS.md`, `make/just verify`, `.env` policy, baseline docs.
2. **Навигация и source of truth**: architecture map, execution flows, local READMEs, ADR index.
3. **Structural alignment**: переименовать/разложить директории, убрать obvious god files, удалить cross-boundary imports.
4. **Executable constraints**: import-linter/layer checks/custom tests/threshold scripts.
5. **Python quality baseline**: `pyproject.toml`, Ruff, type checker, pytest layout, entrypoint separation.
6. **Evals/RAG/observability only if needed**: не внедрять тяжёлые индексы и dashboards, пока нет стабильных docs, verify и boundaries.

## Рекомендуемый итоговый вывод агента

В конце аудита агент должен выдать не просто баллы, а рабочий план:

1. Summary: общий score, 3 главные проблемы, 3 сильные стороны.
2. Score table по секциям A–I.
3. Evidence по каждому `0` и `1`.
4. Red flags: что мешает агентам прямо сейчас.
5. Backlog: `P0/P1/P2/P3`, effort, risk, expected impact.
6. Suggested files to create/update: например `AGENTS.md`, `docs/architecture.md`, `.importlinter`, `justfile`, `docs/adr/0001-...md`.
7. Explicit non-goals: что не стоит чинить сейчас.

Шаблон лежит в `templates/CHECK_REPO_REPORT.template.md`.

## Источники

См. `SOURCES.md`. В методике использованы материалы из приложенного архива и свежие внешние источники 2025–2026 года, включая AGENTS.md/GitHub custom instructions, OpenAI agents guide, Claude/Gemini memory/context docs, OpenTelemetry GenAI semantic conventions, OWASP LLM Top 10, RAG eval docs, Python packaging/Ruff/mypy/import-linter docs, NIST SSDF/SLSA/GitHub push protection и исследования RepoGraph/AI coding architecture.
