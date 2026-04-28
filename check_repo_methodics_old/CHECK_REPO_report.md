# CHECK_REPO_report — Аудит AI-first readiness

> Дата аудита: 2026-04-12
> Репозиторий: defi-research (Python 3.12 monorepo, ~56 KLOC core / ~155 KLOC total)
> Методика: `docs/ai-first-approaches/check_repo_methodics/CHECK_REPO_METHODICS.md`

---

## Сводка

| Секция | Балл | Макс | % |
|--------|------|------|---|
| A. Architecture And Structure | 10 | 12 | 83% |
| B. Codebase Readability | 17 | 18 | 94% |
| C. Docs And Agent Instructions | 12 | 16 | 75% |
| D. Memory And Retrieval | 8 | 14 | 57% |
| E. Verify And Evals | 8 | 16 | 50% |
| F. Observability And Operations | 11 | 16 | 69% |
| G. Governance And Safety | 2 | 10 | 20% |
| H. Python 50–100 KLOC Specific | 12 | 12 | 100% |
| **Итого** | **80** | **114*** | **70%** |

> \* D8 (RAG metrics) = N/A, исключён из максимума (114 вместо 116).

**Интерпретация**: 70% — «usable, but context recovery remains expensive» (диапазон 65–84%). Репозиторий хорошо структурирован архитектурно и код чистый, но слаб в governance, verify-автоматизации и гигиене памяти.

---

## Что у нас хорошо

### Архитектура и код (A+B: 27/30 = 90%)
- **Structural alignment** (A2: 2/2) — директории точно отражают документированную архитектуру. 14 компонентов, orchestrator, shared — всё на своих местах.
- **Execution flows** (A3: 2/2) — 5 подробных flow-описаний: System Flow, FSM, Materialization, Brain DAG, Worker Step Flow.
- **Module boundaries** (A6: 2/2) — zero cross-component imports подтверждён grep-ом. Компоненты общаются только через PostgreSQL, pg_notify, S3 и workspace-файлы.
- **No god files** (B4: 2/2) — нет `utils.py`/`helpers.py`. Утилиты разделены: `logging.py`, `retry.py`, `notifications.py`.
- **Deep modules** (B6: 2/2) — API выражает intent: `service.materialize()`, `service.process_pending()`, `engine.run()`. Никаких `init/prepare/apply/finalize` паттернов.
- **Zero-context survival** (B7: 2/2) — любой компонент понятен за 2 минуты: README → flow.py → service.py.
- **Orchestration ↔ domain separation** (B8: 2/2) — orchestrator/ импортирует только из shared/, никогда из components/.

### Python layout (H: 12/12 = 100%)
- **Unified layout** (H1: 2/2) — все 14 компонентов следуют единому шаблону: `components/<name>/<name>/`.
- **Tools consolidated** (H2: 2/2) — единый `pyproject.toml`: ruff + mypy strict + pytest. Нет конкурирующих конфигов.
- **Strict typing** (H3: 2/2) — mypy `strict = true`, `disallow_untyped_defs = true`. Все публичные функции типизированы.
- **Separation of concerns** (H4: 2/2) — Web/CLI/workers/domain разнесены по отдельным модулям.
- **Infrastructure in docs** (H5: 2/2) — DDL, миграции, очереди, внешние интеграции задокументированы в архитектуре V6.1.
- **Local changes local** (H6: 2/2) — изменение компонента требует чтения только его 3–7 файлов + shared contracts.

### Документация и навигация (C: 12/16 = 75%)
- **AGENTS.md** (C1+C2: 4/4) — 103 строки, навигационный формат. Repo map, Read first, Do not — все три элемента на месте.
- **Progressive disclosure** (C5: 2/2) — явная 4-уровневая модель: AGENTS.md → component READMEs → docs/ARCHITECTURE/ → код.
- **Local READMEs** (C4: 2/2) — все 14 компонентов имеют README.md.

### Observability (F: 11/16 = 69%)
- **Traceable events** (F1: 2/2) — events персистятся в PostgreSQL, дедуплицируются по idempotency key, structured logging.
- **Structured logs** (F2: 2/2) — structlog JSONRenderer, ISO 8601 timestamps, LLMResponse трекает tokens/cost/latency.
- **Content-level observability** (F3: 2/2) — 5-мерные security scores (0–10), confidence, data_completeness, safety_rating с порогами.
- **Human review by thresholds** (F4: 2/2) — quality loop-back с revision_targets и структурированным feedback по шагам.
- **Prompt versioning** (F7: 2/2) — промпты в Git → версия в БД → traces в Langfuse. Полная lineage-цепочка.

### Eval contract (E6: 2/2)
- `scripts/eval_brain.py` (~600 строк): 6 категорий проверок, SAFETY_THRESHOLDS, exit code contract. Покрыт тестами.

---

## Что стоит доработать

### Подробная таблица с оценками

Каждый пункт оценивается по двум шкалам:
- **Полезность** (1–10): насколько это улучшит работу AI-агентов и команды.
- **Сложность** (1–10): насколько трудоёмко реализовать (10 = очень трудоёмко).
- **KPI** = Полезность − Сложность. Чем выше KPI, тем выгоднее делать первым.

---

#### G3: CI pipeline — Merge не блокируется при провале проверок
**Текущий Score: 0/2** | **Priority: P1**

Evidence: в корне нет `.github/workflows/` — репозиторный CI отсутствует. По локальным файлам нельзя подтвердить наличие branch protection. При этом в `docs/IMPLEMENTATION/05_release_and_product.md` быстрый CI уже выделен как stage 1, а полный CI остаётся задачей `5.1`.

| Полезность | Сложность | KPI |
|-----------|-----------|-----|
| **9** | **3** | **+6** |

Рекомендация: сейчас добавить только быстрый CI: `ruff check .`, `mypy .`, `pytest tests/ -m "not integration"`. Полный CI (integration, golden dataset, required checks и branch protection) — позже, вместе с задачей `5.1`.

---

#### G4: Infrastructure gates — все ограничения только текстовые
**Текущий Score: 0/2** | **Priority: P2**

Evidence: все правила из AGENTS.md "Do not" — только prose. Правило «не импортировать между компонентами» нарушено в `components/backend/backend/rest/admin.py`. Нет pre-commit hooks.

| Полезность | Сложность | KPI |
|-----------|-----------|-----|
| **7** | **4** | **+3** |

Рекомендация: после быстрого CI добавить минимальный `.pre-commit-config.yaml`: `ruff` + `detect-secrets`. `mypy` лучше оставить в CI, чтобы не замедлять локальный цикл обратной связи. Расширять hooks — только после стабилизации базового verify-контура.

---

#### G5: Автоматизация архитектурных границ
**Текущий Score: 0/2** | **Priority: P1**

Evidence: нет `import-linter`, нет boundary tests для module boundaries. Правило cross-component imports enforced только convention, при этом уже есть известный обход в `components/backend/backend/rest/admin.py`.

| Полезность | Сложность | KPI |
|-----------|-----------|-----|
| **9** | **6** | **+3** |

Рекомендация: не включать как блокирующее правило прямо сейчас. Сначала закрыть задачи `R.1-R.3` по очистке границ, затем ввести `import-linter` и boundary tests, и только после этого подключать их в CI.

---

#### E1: Единая команда verify
**Текущий Score: 0/2** | **Priority: P0**

Evidence: нет Makefile, tox.ini, noxfile. Разработчик должен помнить и запускать `ruff check .`, `mypy .`, `pytest tests/` по отдельности.

| Полезность | Сложность | KPI |
|-----------|-----------|-----|
| **8** | **1** | **+7** |

Рекомендация: создать `Makefile` с таргетами `lint`, `typecheck`, `test`, `verify` (= lint + typecheck + test).

---

#### E4: Runbooks и DEBUG.md
**Текущий Score: 0/2** | **Priority: P2**

Evidence: нет `DEBUG.md`, `TROUBLESHOOTING.md`, `runbooks/`. `08_observability_ops_security.md` описывает условия алертов, но не процедуры восстановления.

| Полезность | Сложность | KPI |
|-----------|-----------|-----|
| **2** | **5** | **−3** |

Рекомендация: не включать в текущую волну улучшения репо. Вернуться к runbooks на этапе продакшн-деплоя (`5.2-5.3`) или при появлении повторяющейся операционной боли.

---

#### D2: Git свободен от scratchpad-файлов
**Текущий Score: 0/2** | **Priority: P1**

Evidence: в Git остаются архивные и planning-артефакты, которые создают дополнительный retrieval-шум. При этом часть legacy TODO уже переведена в frozen/pointer mode и больше не является актуальным backlog.

| Полезность | Сложность | KPI |
|-----------|-----------|-----|
| **3** | **2** | **+1** |

Рекомендация: не включать в приоритетный план улучшения репо. Провести очистку вручную и отдельно, без смешивания с verify/CI/SSOT задачами.

!!! Комментарий заказчика для ИИ: Не трогай, это я сам сделаю. !!!

---

#### D6: Symbol index / call graph
**Текущий Score: 0/2** | **Priority: P3**

Evidence: нет graph-based navigation. При 155K LOC agents, работающие cross-boundary (orchestrator ↔ shared ↔ components), выиграли бы от call graph.

| Полезность | Сложность | KPI |
|-----------|-----------|-----|
| **4** | **6** | **−2** |

Рекомендация: отложить. При текущей структуре grep + file tree + docs достаточно для большинства задач.

---

#### F5: Incident playbooks для AI-специфичных сбоев
**Текущий Score: 0/2** | **Priority: P3**

Evidence: нет `docs/runbooks/`, `docs/playbooks/`. Alert conditions определены, но нет resolution procedures для галлюцинаций, роста стоимости, injection, retrieval drift.

| Полезность | Сложность | KPI |
|-----------|-----------|-----|
| **2** | **5** | **−3** |

Рекомендация: не включать в текущую волну улучшения репо. Это уже больше про операционную модель и AI-инциденты в продакшне, чем про гигиену репозитория.

!!! Комментарий заказчика для ИИ: Не трогай, это уже бизнес-логика, а не улучшение репо. !!!

---

#### F8: OTel GenAI semantic conventions
**Текущий Score: 0/2** | **Priority: P3**

Evidence: используется Langfuse (vendor-specific). Нет opentelemetry-* пакетов. Нет gen_ai.* атрибутов.

| Полезность | Сложность | KPI |
|-----------|-----------|-----|
| **3** | **7** | **−4** |

Рекомендация: отложить. Langfuse покрывает текущие потребности. OTel GenAI semconv — вопрос масштабирования.

---

#### A4: ADR / журналы решений
**Текущий Score: 1/2** | **Priority: P1**

Evidence: ключевые решения уже зафиксированы в `ARCHITECTURE_V6.1_index.md` (12 done, 3 MVP limitations, 5 not done) и фактически работают как облегчённый журнал решений. Отдельного каталога `docs/adr/` нет.

| Полезность | Сложность | KPI |
|-----------|-----------|-----|
| **4** | **4** | **0** |

Рекомендация: не выносить решения в `docs/adr/` только ради формы. Пока достаточно текущего блока в `ARCHITECTURE_V6.1_index.md`. Если независимых решений станет больше, логичнее завести `docs/ARCHITECTURE/decision_log.md` или `docs/ARCHITECTURE/decisions/`.

!!! Комментарий заказчика для ИИ: Чем docs/adr/ лучше текущей папки? !!!

---

#### A5: Executable architecture constraints
**Текущий Score: 1/2** | **Priority: P1**

Evidence: mypy strict + ruff + `tests/test_architecture_docs.py` уже есть, но нет import-linter и автоматически проверяемых в CI boundary checks. Пересекается с G5.

| Полезность | Сложность | KPI |
|-----------|-----------|-----|
| **6** | **5** | **+1** |

Рекомендация: закрывать через G5 после очистки архитектурных границ, а не как отдельный срочный проект.

---

#### B9: Объективные пороги сложности
**Текущий Score: 1/2** | **Priority: P1**

Evidence: ruff не включает C90/mccabe. 6 файлов >500 строк (recovery.py — 1623 строки). Cyclomatic complexity не измеряется.

| Полезность | Сложность | KPI |
|-----------|-----------|-----|
| **5** | **4** | **+1** |

Рекомендация: не делать сразу блокирующее правило. Сначала ввести complexity-метрики только как отчёт/метрику или для изменённых файлов, затем ужесточать пороги постепенно.

---

#### C3: README.md — ссылки на устаревшую архитектуру
**Текущий Score: 1/2** | **Priority: P1**

Evidence: README.md ссылается на `ARCHITECTURE_V5.md` и `IMPLEMENTATION_PLAN_V2.md`, тогда как SSOT — V6.1. Нет команд lint/test.

| Полезность | Сложность | KPI |
|-----------|-----------|-----|
| **8** | **1** | **+7** |

Рекомендация: обновить ссылки на V6.1 и canonical implementation docs, добавить короткую секцию verification commands.

---

#### C6: Single source of truth
**Текущий Score: 1/2** | **Priority: P1**

Evidence: SSOT объявлен, но `README.md`, `docs/AI_PRACTICES.md`, `components/brain/README.md` и часть `.planning/codebase/*` всё ещё ведут на V5 или старое состояние.

| Полезность | Сложность | KPI |
|-----------|-----------|-----|
| **8** | **1** | **+7** |

Рекомендация: пройтись grep-ом по ссылкам на V5/V6 и обновить все верхнеуровневые точки входа до V6.1 и `docs/IMPLEMENTATION/*`. Cleanup legacy TODO-файлов не включать в эту задачу.

!!! Комментарий заказчика для ИИ: Касательно "Консолидировать TODO-файлы" - опять же не трогай, это я сам сделаю. !!!

---

#### C7: Doc-gardening
**Текущий Score: 1/2** | **Priority: P2**

Evidence: `scripts/architecture_docs.py` валидирует только `docs/ARCHITECTURE/`. Нет link checker, нет stale-doc detection для остальных docs.

| Полезность | Сложность | KPI |
|-----------|-----------|-----|
| **4** | **4** | **0** |

Рекомендация: добавить link checker + age-based flags. Расширить doc validation на весь docs/.

---

#### C8: AI-instruction files — дублирование
**Текущий Score: 1/2** | **Priority: P1**

Evidence: `AGENTS.md` ≈ `CLAUDE.md` (идентичны с точностью до whitespace). `.planning/codebase/` (7 файлов) дублирует факты из AGENTS.md. Три места для обновления.

| Полезность | Сложность | KPI |
|-----------|-----------|-----|
| **4** | **2** | **+2** |

Рекомендация: не удалять и не симлинковать `CLAUDE.md` автоматически, пока не подтверждено поведение загрузки контекста у агентов. Если нужны оба файла, лучше ввести явный статус зеркала и проверку синхронизации. Основной риск здесь — рассинхрон, а не сам факт двух файлов.

!!! Комментарий заказчика для ИИ: CLAUDE.md и AGENTS.md - осознанно продублированы для разных агентов. Можно сделать симлинк, если это точно не скажется на подгузку файла в контекст агента. !!!

---

#### D1: Устойчивые знания — ADR, runbooks, post-mortem
**Текущий Score: 1/2** | **Priority: P1**

Evidence: архитектурная и implementation-память сильные, но в одном пункте смешаны разные классы артефактов: architectural decision memory и operational memory. Нет отдельного ADR-каталога, runbooks, post-mortem и CHANGELOG.

| Полезность | Сложность | KPI |
|-----------|-----------|-----|
| **3** | **5** | **−2** |

Рекомендация: не делать bundled-задачу сейчас. Для архитектурной памяти достаточно текущего облегчённого журнала решений; к операционной памяти (`runbooks`, `post-mortem`, `CHANGELOG`) имеет смысл возвращаться после появления реального релизного и операционного цикла.

!!! Комментарий заказчика для ИИ: Опять же runbooks - не делаем. Опять же вопрос про путь. И насчёт CHANGELOG - нужен ли он, если мы в docs/IMPLEMENTATION/00_status.md отмечаем сделанное? !!!

---

#### D5: Retrieval ladder — соответствие размеру кодовой базы
**Текущий Score: 1/2** | **Priority: P3**

Evidence: при 155K LOC retrieval чисто навигационный (AGENTS.md → docs → code). Для этого размера нужна как минимум hybrid search. `arch:block` markup — хорошая основа, но не подключён к поиску.

| Полезность | Сложность | KPI |
|-----------|-----------|-----|
| **4** | **7** | **−3** |

Рекомендация: отложить. Текущая навигационная структура справляется. Arch:block markup можно подключить позже.

---

#### E2: Быстрый локальный цикл обратной связи
**Текущий Score: 1/2** | **Priority: P1**

Evidence: конфиги ruff/mypy/pytest есть, маркеры `integration`/`slow` определены. Но нет скриптовых таргетов. Пересекается с E1.

| Полезность | Сложность | KPI |
|-----------|-----------|-----|
| **8** | **1** | **+7** |

Рекомендация: решать вместе с E1 — короткими таргетами `lint`, `typecheck`, `unit/test` и одним верхнеуровневым `verify`.

---

#### E3: Скрипты важнее текста — первичная настройка не скриптирована
**Текущий Score: 1/2** | **Priority: P1**

Evidence: `scripts/` содержит 12+ операционных скриптов (run_protocol.py, eval_brain.py). Но начальная настройка (pip install, alembic, .env) всё ещё описана текстом в README.

| Полезность | Сложность | KPI |
|-----------|-----------|-----|
| **3** | **3** | **0** |

Рекомендация: не включать в текущую волну улучшения репо. Вернуться к `make setup` / `scripts/setup.sh`, если первичная настройка действительно станет узким местом после E1/G3.

!!! Комментарий заказчика для ИИ: Не трогай, это уже бизнес-логика, а не улучшение репо. !!!

---

#### E5: Architecture tests для module boundaries
**Текущий Score: 1/2** | **Priority: P1**

Evidence: `tests/test_architecture_docs.py` и часть контрактных тестов уже есть, но нет import boundary tests. Пересекается с G5/A5.

| Полезность | Сложность | KPI |
|-----------|-----------|-----|
| **9** | **6** | **+3** |

Рекомендация: см. G5; включать после очистки архитектурных границ, а не раньше.

---

#### E7: Golden datasets — нет regression gate
**Текущий Score: 1/2** | **Priority: P2**

Evidence: `defres-data/` содержит рабочие workspace-ы с scores/citations. Нет автоматического сравнения с эталоном при изменениях.

| Полезность | Сложность | KPI |
|-----------|-----------|-----|
| **2** | **6** | **−4** |

Рекомендация: не включать в текущую волну улучшения репо. Вернуться к regression gate после задачи `2.3.1` и появления eval-контура релизного уровня.

!!! Комментарий заказчика для ИИ: Не трогай, это уже бизнес-логика, а не улучшение репо. !!!

---

#### F6: Post-mortems → возврат знаний в проект
**Текущий Score: 1/2** | **Priority: P2**

Evidence: review→fix→docs update происходит органически (видно по git log). Нет формализованного процесса post-mortem.

| Полезность | Сложность | KPI |
|-----------|-----------|-----|
| **2** | **3** | **−1** |

Рекомендация: не включать в текущую волну улучшения репо. Формализовать post-mortem цикл позже, когда появятся инциденты в продакшне или более широкий инженерный контур.

---

#### G1: Политика secrets
**Текущий Score: 1/2** | **Priority: P1**

Evidence: `.env.example` есть, `.gitignore` блокирует `.env`. Нет документированной политики, нет secret scanning (detect-secrets, gitleaks).

| Полезность | Сложность | KPI |
|-----------|-----------|-----|
| **8** | **2** | **+6** |

Рекомендация: добавить короткую политику в AGENTS.md и secret scanning с базовым файлом исключений, чтобы избежать ложных срабатываний на `.env.example`.

---

#### G2: Критические зоны — отметки в коде
**Текущий Score: 1/2** | **Priority: P1**

Evidence: AGENTS.md "Do not" + `08-security-and-governance.md` 4-tier risk classification. Но в Python-коде нет `# CRITICAL:` / `# DANGER:` маркеров.

| Полезность | Сложность | KPI |
|-----------|-----------|-----|
| **3** | **2** | **+1** |

Рекомендация: использовать выборочно и только на действительно опасных путях. Массово размечать код комментариями не стоит.

---

## Ранжирование по KPI

| Ранг | Пункт | Полезность | Сложность | KPI | Описание |
|------|-------|-----------|-----------|-----|----------|
| 1 | **E1** | 8 | 1 | **+7** | Единая команда verify |
| 2 | **E2** | 8 | 1 | **+7** | Быстрый локальный цикл (вместе с E1) |
| 3 | **C3** | 8 | 1 | **+7** | Починить README как точку входа |
| 4 | **C6** | 8 | 1 | **+7** | Починить SSOT и устаревшие ссылки |
| 5 | **G1** | 8 | 2 | **+6** | Политика secrets + сканирование |
| 6 | **G3** | 9 | 3 | **+6** | Только быстрый CI сейчас |
| 7 | **G4** | 7 | 4 | **+3** | Минимальный pre-commit после быстрого CI |
| 8 | **G5/A5/E5** | 9 | 6 | **+3** | Автоматизация границ после очистки архитектуры |
| 9 | **C8** | 4 | 2 | **+2** | Контроль рассинхрона между AI-instruction файлами |
| 10 | **B9** | 5 | 4 | **+1** | Пороги сложности сначала только как метрика |
| 11 | **G2** | 3 | 2 | **+1** | Выборочные CRITICAL markers |
| 12 | **D2** | 3 | 2 | **+1** | Ручная очистка, вне приоритетного плана |
| 13 | **A4** | 4 | 4 | **0** | Формализация ADR позже, если вырастет объём решений |
| 14 | **C7** | 4 | 4 | **0** | Doc-gardening позже |
| 15 | **E3** | 3 | 3 | **0** | Скрипт первичной настройки только при реальной боли онбординга |
| 16 | **F6** | 2 | 3 | **−1** | Post-mortem process позже |
| 17 | **D1** | 3 | 5 | **−2** | Не смешивать decision memory и operational memory |
| 18 | **D6** | 4 | 6 | **−2** | Symbol graph (отложить) |
| 19 | **D5** | 4 | 7 | **−3** | Hybrid search (отложить) |
| 20 | **E4/F5** | 2 | 5 | **−3** | Runbooks и AI-incident playbooks не сейчас |
| 21 | **E7** | 2 | 6 | **−4** | Regression gate позже, после `2.3.1` |
| 22 | **F8** | 3 | 7 | **−4** | OTel GenAI (отложить) |

---

## Рекомендуемый порядок исправлений

### Волна 1 — самое актуальное сейчас

1. **E1 + E2**: единая команда `verify` и короткие локальные таргеты `lint` / `typecheck` / `test`
2. **C3 + C6**: обновить `README.md` и другие верхнеуровневые точки входа до V6.1 и `docs/IMPLEMENTATION/*` (В C6 - "Консолидировать TODO-файлы" - не надо!)
3. **G1**: добавить минимальную политику secrets и secret scanning с базовым файлом исключений
4. **G3**: внедрить только быстрый CI (`ruff`, `mypy`, `pytest -m "not integration"`)

### Волна 2 — автоматическое усиление правил после базовой стабилизации

5. **G4**: минимальный `.pre-commit-config.yaml` (`ruff` + `detect-secrets`)
6. **G5/A5/E5**: import-linter и boundary tests после задач `R.1-R.3`
7. **B9**: complexity thresholds сначала только как метрика

### Волна 3 — позже, если появится реальная потребность

8. **C8**: ввести контроль синхронизации для `AGENTS.md` / `CLAUDE.md`, не меняя схему загрузки контекста вслепую
9. **C7**: doc-gardening и link checker
10. **A4**: формальный журнал решений только если решений станет заметно больше
11. **G2**: выборочные CRITICAL markers на реально опасных путях
12. **E3**: скрипт первичной настройки, если настройка текстом начнёт мешать

### Вне текущего плана улучшения репо

- **D2**: ручная очистка архивных и planning-артефактов — отдельно, без включения в текущие волны
- **D1**: не собирать сейчас в одну задачу ADR + runbooks + post-mortem + CHANGELOG
- **E4 / F5**: runbooks и AI-incident playbooks — позже, вместе с практиками продакшна и релизов
- **F6**: формальный post-mortem цикл — позже
- **E7**: regression gate — позже, после `2.3.1` и eval-контура релизного уровня
- **D5**: hybrid search — текущая навигация справляется
- **D6**: symbol/call graph — при хорошей структуре не критично
- **F8**: OTel GenAI — Langfuse покрывает потребности

---

## Полный аудит-лист

### A. Architecture And Structure

| ID | Priority | Проверка | Score | Evidence |
|----|----------|----------|-------|----------|
| A1 | P1 | Краткая карта системы | **2** | 11 модульных docs в ARCHITECTURE V6.1, index (143 строки), pipeline one-liner в README, 5 flow-диаграмм в docs |
| A2 | P0 | Структура директорий ↔ архитектура | **2** | 14 компонентов, orchestrator/, shared/ — точно соответствуют 06_components.md |
| A3 | P1 | Описание 3–7 execution flows | **2** | 5 flows: System Flow, FSM Transitions, Materialization, Brain DAG, Worker Step Flow |
| A4 | P1 | ADR или decision log | **1** | Решения встроены в ARCHITECTURE_V6.1_index.md (12 done, 3 MVP, 5 not done). Нет формального docs/adr/ |
| A5 | P1 | Constraints enforced tools/tests | **1** | mypy strict + ruff + test_architecture_docs.py. Нет import-linter, нет CI |
| A6 | P1 | Module boundaries | **2** | `__all__` в shared/, `_private` naming, границы в целом читаются по коду, но есть локальный обход в `components/backend/backend/rest/admin.py` |

### B. Codebase Readability

| ID | Priority | Проверка | Score | Evidence |
|----|----------|----------|-------|----------|
| B1 | P0 | Доменные модули читаются по дереву | **2** | Имена самодокументирующиеся: materializer, syncer, digest_engine. 14 README |
| B2 | P1 | Public/private API separation | **2** | `__all__` в shared/ (4–62 экспорта). `_pool`, `_init_connection()` — private naming |
| B3 | P1 | Side effects локализованы | **2** | Все `__init__.py` чистые. Zero monkey-patching, zero metaclass. Singletons lock-protected |
| B4 | P0 | Нет god files | **2** | Нет utils.py/helpers.py. Утилиты разделены: logging.py, retry.py, notifications.py |
| B5 | P1 | Honest contracts | **2** | mypy strict + все публичные функции типизированы + docstrings + Pydantic models |
| B6 | P1 | Deep modules | **2** | `service.materialize()`, `engine.run()` — semantic API без lifecycle scaffolding |
| B7 | P1 | Zero-context survival | **2** | Materializer, Audit Reader — оба понятны за 2 мин cold read |
| B8 | P1 | Orchestration ↔ domain | **2** | orchestrator/ импортирует только shared/, components содержат только domain logic |
| B9 | P1 | Пороги размера/сложности | **1** | ruff без C90/mccabe. 6 файлов >500 строк (recovery.py: 1623). CC не измеряется |

### C. Docs And Agent Instructions

| ID | Priority | Проверка | Score | Evidence |
|----|----------|----------|-------|----------|
| C1 | P0 | AGENTS.md как карта | **2** | 103 строки, навигационный формат, не энциклопедия |
| C2 | P1 | Структура, документы, запреты | **2** | Repo map (8 dirs), Read first (7 docs), Do not (6 запретов) |
| C3 | P1 | README.md — что, как, verify | **1** | Хорошая структура, но ссылки на V5 вместо V6.1, нет команд lint/test |
| C4 | P2 | Local README в крупных dirs | **2** | 14/14 компонентов. Нет в shared/ и tests/ (support dirs) |
| C5 | P1 | Progressive disclosure | **2** | 4-уровневая модель, задокументирована в AI_PRACTICES.md |
| C6 | P1 | Single source of truth | **1** | SSOT объявлен, но README/AI_PRACTICES/brain README и часть `.planning/codebase/*` всё ещё ссылаются на V5 или старое состояние |
| C7 | P2 | Doc-gardening | **1** | architecture_docs.py валидирует ARCHITECTURE/. Нет link checker и детекции устаревших docs для остальных разделов |
| C8 | P1 | AI-instruction files | **1** | AGENTS.md ≈ CLAUDE.md (дубликат). .planning/codebase/ (840 строк) — третий слой дублирования |

### D. Memory And Retrieval

| ID | Priority | Проверка | Score | Evidence |
|----|----------|----------|-------|----------|
| D1 | P1 | Устойчивые знания | **1** | Архитектурная и implementation-память сильные, но decision memory и operational memory пока не разведены по разным уровням зрелости |
| D2 | P2 | Нет scratchpad в Git | **0** | В Git остаются архивные и planning-артефакты; часть legacy TODO уже frozen/pointer-mode, но retrieval-шум всё ещё есть |
| D3 | P1 | Map → document → code | **2** | Трёхуровневая модель, architecture index с reading paths по ролям |
| D4 | P1 | Agent-directed retrieval | **2** | AGENTS.md направляет к конкретным файлам, AI_PRACTICES явно предупреждает о context stuffing |
| D5 | P3 | Retrieval ladder ↔ размер | **1** | 155K LOC, чисто навигационный retrieval. arch:block есть, но не подключён к поиску |
| D6 | P3 | Symbol/call graph | **0** | Нет graph-based navigation tools |
| D7 | P1 | Long context ≠ архитектура | **2** | Docs разбиты на 10 модульных файлов, не монолит. Монолит — build artifact с warning |
| D8 | P2 | RAG: relevance + faithfulness | **N/A** | Нет RAG в проекте. File-based retrieval уместен для текущей архитектуры |

### E. Verify And Evals

| ID | Priority | Проверка | Score | Evidence |
|----|----------|----------|-------|----------|
| E1 | P0 | Единая команда verify | **0** | Нет Makefile, tox, nox, justfile. ruff/mypy/pytest запускаются раздельно |
| E2 | P1 | Fast local loop | **1** | Конфиги и test markers есть, но нет коротких локальных таргетов и единой точки входа |
| E3 | P1 | Scripts over prose | **1** | 12+ ops scripts есть, но начальный setup всё ещё описан текстом в README |
| E4 | P2 | Runbooks проверены | **0** | Нет DEBUG.md, troubleshooting, runbooks/. Alert conditions без resolution procedures |
| E5 | P1 | Architecture tests | **1** | `test_architecture_docs.py` и часть contract tests уже есть, но нет import boundary tests и автоматически проверяемых в CI fitness checks |
| E6 | P2 | Eval contract | **2** | eval_brain.py (600 строк): 6 check categories, SAFETY_THRESHOLDS, exit code contract, тесты |
| E7 | P2 | Golden datasets versioned | **1** | `defres-data/` с workspace-ами уже есть, но regression gate — это отдельная product/eval задача, а не гигиена репозитория |
| E8 | P2 | Remediation instructions | **2** | Validators: field + expected + actual. quality_review loop-back как remediation |

### F. Observability And Operations

| ID | Priority | Проверка | Score | Evidence |
|----|----------|----------|-------|----------|
| F1 | P2 | Agent runs traceable | **2** | Events в PostgreSQL, idempotency key, structured logging с event_id |
| F2 | P2 | Structured logs + metrics | **2** | structlog JSONRenderer, DB metrics (pipeline_state, transitions), LLMResponse cost tracking |
| F3 | P3 | Content-level observability | **2** | 5-dim security scores, confidence, data_completeness, safety_rating с порогами |
| F4 | P3 | Human review by thresholds | **2** | Quality loop-back: verdict → revision_targets → revision_feedback_by_step |
| F5 | P3 | Incident playbooks | **0** | Alert conditions определены, но формальные AI-incident playbooks пока отсутствуют и относятся скорее к операционной зрелости |
| F6 | P2 | Post-mortems → feedback | **1** | Органический review→fix→docs update есть, но формальный post-mortem процесс пока не нужен как отдельный контур улучшения репо |
| F7 | P2 | Prompts versioned + traced | **2** | Git → prompt_version в DB → Langfuse traces. Полная lineage |
| F8 | P3 | OTel GenAI conventions | **0** | Langfuse only, vendor-specific. Нет OTel пакетов |

### G. Governance And Safety

| ID | Priority | Проверка | Score | Evidence |
|----|----------|----------|-------|----------|
| G1 | P1 | Secrets + .env policy | **1** | `.env.example` и `.gitignore` есть, но нет явной политики и secret scanning с базовым файлом исключений |
| G2 | P1 | Critical zones marked | **1** | AGENTS.md "Do not" + 4-tier risk docs уже есть; в коде нет точечных CRITICAL markers на действительно опасных путях |
| G3 | P1 | Merge blocked on failures | **0** | Корневого CI нет; быстрый CI стоит внедрить сейчас, полный CI релизного уровня — позже в рамках `5.1` |
| G4 | P2 | Infrastructure > prose | **0** | Критичные правила в основном text-only; известный обход границ в `admin.py` показывает, что автоматическое enforcement пока слабое |
| G5 | P1 | Boundary automation | **0** | Нет import-linter и автоматизации границ; делать имеет смысл после задач `R.1-R.3` по очистке границ |

### H. Python 50–100 KLOC Specific

| ID | Priority | Проверка | Score | Evidence |
|----|----------|----------|-------|----------|
| H1 | P1 | Unified layout | **2** | 14 компонентов по единому шаблону. shared imports consistent |
| H2 | P1 | Tools in pyproject.toml | **2** | Единый pyproject.toml: ruff + mypy strict + pytest. Нет конкурирующих конфигов |
| H3 | P1 | Typing mandatory | **2** | mypy strict + disallow_untyped_defs. Pydantic для всех shared shapes |
| H4 | P1 | Entrypoint separation | **2** | Web/CLI/workers/domain в разных модулях |
| H5 | P2 | Infra in docs | **2** | DDL в appendix, data model docs, Alembic migrations, events/queues documented |
| H6 | P0 | Local changes local | **2** | Компонент = 3–7 файлов + shared contracts. Минимальный coupling |

---

## Ключевые выводы

**Сильные стороны:**
- Архитектура и код на уровне 90%+ — редкость для проекта такого размера
- Python layout идеален (100%) — единый шаблон, strict typing, consolidated tooling
- Observability для AI-системы выше среднего — content-level scoring, prompt lineage, threshold-based review
- Progressive disclosure и agent-directed retrieval продуманы и задокументированы

**Критические пробелы:**
- **Governance = 20%** — главная слабость. Нет CI, нет автоматического enforcement, нет автоматизации. Все правила — только текст
- **Verify = 50%** — нет единой команды и быстрой локальной точки входа. При отличных tool configs они не wired together
- **Memory hygiene = 57%** — главная проблема не в отсутствии RAG, а в устаревающем контексте и дополнительных слоях дублирования/дрейфа

**Стратегия:**
Волна 1 должна закрыть самые дешёвые и самые прикладные вещи: единый verify, быстрый локальный цикл, очистку SSOT, политику secrets и быстрый CI. Волна 2 — автоматическое усиление правил после стабилизации границ. Процессные пункты вроде runbooks, post-mortem и regression gates лучше не смешивать с текущей волной улучшения репо.
