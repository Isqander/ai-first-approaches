# 07 — Самомодификация (Self-Modification)

> Как AI-система эволюционирует: от ручных улучшений к автоматическому self-improvement.

---

## 1. Зачем самомодификация

AI-система — не статическое ПО. Контекст меняется: новые модели, новые API, новые паттерны. Система должна **адаптироваться** без полного переписывания.

Три уровня:
1. **Knowledge evolution** — обновление знаний (документы, skills, workflows)
2. **Operational evolution** — оптимизация процессов (routing, prompts, tool selection)
3. **Structural evolution** — изменение архитектуры (новые модули, рефакторинг)

---

## 2. Два режима

### 2.1 Полуавтоматический (фазы 1-2)

```
evolution-steward (weekly cron)
   ├─ Анализирует:
   │   ├─ Event logs (ошибки, timeouts, retries)
   │   ├─ OperationLog (bottlenecks, повторяющиеся паттерны)
   │   ├─ Eval results (деградация качества)
   │   ├─ Обновления repos из examples/ (новые идеи)
   │   └─ Обновления зависимостей (Stoneforge npm)
   ├─ Формирует:
   │   └─ Document «Evolution Backlog — Week N»
   │       ├─ Проблемы (ранжированные по impact)
   │       ├─ Возможности (новые features, optimizations)
   │       └─ Технический долг
   └─ Отправляет в Telegram / Frontend
```

Человек ревьюит backlog → выбирает задачи → запускает реализацию.

### 2.2 Автоматический с апрувом (фаза 3+)

```
evolution-steward обнаруживает → формулирует Task
   │
   ├─ LOW risk (knowledge, docs):
   │   → Worker → реализация → auto-merge → notify
   │
   ├─ MEDIUM risk (skills, workflows):
   │   → Worker → реализация → MergeSteward (CI) → notify
   │
   └─ HIGH/CRITICAL risk (code, config, CONSTITUTION):
       → Worker → реализация → ApprovalService
           → Telegram: описание изменений + diff
           → Человек: approve / reject / modify
           → If approved → merge
```

---

## 3. Области самомодификации

### 3.1 Skills и Workflows

| Что | Как | Risk |
|-----|-----|------|
| Новый skill (шаблон ответа) | Анализ повторяющихся задач → шаблонизация | LOW |
| Оптимизация workflow | Анализ длительных/дорогих pipelines | MEDIUM |
| Новый playbook | Обнаружение паттерна → playbook YAML | MEDIUM |

### 3.2 Routing и Prompts

| Что | Как | Risk |
|-----|-----|------|
| Новые intent patterns | Анализ misclassified messages | LOW |
| Prompt optimization | Eval-driven: изменение → eval → сравнение | MEDIUM |
| Новые event handlers | Повторяющийся паттерн → детерминированный handler | MEDIUM |

### 3.3 Tools и MCP

| Что | Как | Risk |
|-----|-----|------|
| Новый MCP server | Повторяющееся использование external API → обёртка | MEDIUM |
| Оптимизация tool selection | Анализ: какие tools чаще успешны для каких задач | LOW |
| Self-created tools | Паттерн обнаружен → MCP server → регистрация → docs | MEDIUM-HIGH |

### 3.4 Документация

| Что | Как | Risk |
|-----|-----|------|
| Обновление docs | Doc-gardening: актуализация устаревшего | LOW |
| Новые ADRs | Архитектурное решение принято → зафиксировать | LOW |
| README.md updates | Структура изменилась → обновить | LOW |

### 3.5 Код системы

| Что | Как | Risk |
|-----|-----|------|
| Рефакторинг | Technical debt analysis → proposal | HIGH |
| Новый модуль | Требования изменились → дизайн + реализация | HIGH |
| Обновление зависимостей | update-steward: patch auto, minor/major HITL | MEDIUM-HIGH |

---

## 4. Self-Created Tools

### 4.1 Процесс

```
1. Агент замечает повторяющийся паттерн:
   "Я часто делаю X → вызываю Y → парсю Z → форматирую W"

2. Анализирует:
   - Частота (>5 раз за неделю?)
   - Стабильность (API не меняется?)
   - Автоматизируемость (можно ли без LLM?)

3. Реализует:
   - MCP server (если нужен access из разных агентов)
   - Скрипт (если one-off)
   - Skill/Workflow (если нужна LLM-обработка)

4. Интеграция:
   - Регистрация в MCP Gateway
   - Документирование (tool description для agents)
   - Тестирование

5. Обновление роутинга:
   - Director-Router узнаёт про новый инструмент
```

### 4.2 SafeClaw принцип

Прежде чем создавать AI-инструмент → проверить: можно ли решить **детерминированно**?

```
✅ Парсинг JSON → скрипт (без LLM)
✅ RSS polling → cron + HTTP (без LLM)
✅ File formatting → template engine (без LLM)
❌ Summarization → нужен LLM
❌ Classification → нужен LLM (или ML model)
```

---

## 5. Entropy и Garbage Collection

### 5.1 Проблема энтропии

Со временем в репозитории накапливается:
- Dead code (не вызывается)
- Устаревшая документация
- Заброшенные TODO/FIXME
- Дублирующиеся Skills/Workflows
- Неиспользуемые зависимости
- Неактуальные тесты

### 5.2 Garbage Collection Runs

Периодические «уборки» (по аналогии с GC в runtime):

```
GC Steward (monthly cron)
   ├─ Dead code detection:
   │   └─ Анализ imports/exports → unused exports → report
   ├─ Documentation staleness:
   │   └─ Files not updated >90 days → review queue
   ├─ Dependency audit:
   │   └─ npm outdated + security audit → report
   ├─ Test coverage:
   │   └─ Uncovered critical paths → test creation tasks
   └─ Duplicate detection:
       └─ Similar documents/skills → merge candidates
```

### 5.3 Golden Principles для GC

1. **Never delete without backup** — Git помнит всё, но backup intent
2. **Report before act** — GC формирует отчёт, не удаляет молча
3. **Human approval for structural changes** — удаление модулей = HIGH risk
4. **Measure entropy** — трек количества TODO, dead code %, doc staleness

---

## 6. Update Strategy (Stoneforge)

### 6.1 Smart Update Steward

```
update-steward (weekly cron)
   ├─ npm outdated @stoneforge/*
   ├─ Читает changelog каждого обновления
   ├─ Классифицирует:
   │   ├─ patch (bug fixes) → auto-update + test
   │   ├─ minor (new features) → notify + test
   │   └─ major (breaking changes) → human review + migration plan
   └─ Для auto-updates:
       ├─ npm update
       ├─ npm run verify
       ├─ If pass → commit + push
       └─ If fail → rollback + report
```

### 6.2 Stoneforge-specific

Stoneforge может обновить CLI agent configurations, prompts, workflows. Update Steward должен понимать:
- Какие prompts изменились → может повлиять на routing
- Какие API изменились → может сломать AI OS Shell
- Какие defaults изменились → может изменить behavior

---

## 7. Ограничения самомодификации

| Ограничение | Причина |
|------------|--------|
| **CONSTITUTION.md** — только с approval | Идентичность системы |
| **Security rules** — только с approval | Безопасность |
| **Architectural changes** — только с approval | Долгосрочные последствия |
| **External APIs** — не создавать без review | Стоимость, rate limits |
| **Git push / deploy** — только с approval | Необратимость |

---

## 8. Метрики эволюции

| Метрика | Что показывает |
|---------|---------------|
| **Tasks auto-resolved / week** | Сколько задач система решила без человека |
| **New skills created / month** | Скорость обучения |
| **Prompt changes / month** | Активность оптимизации |
| **Eval score trend** | Качество растёт или деградирует? |
| **Technical debt score** | Здоровье кодовой базы |
| **Doc freshness %** | Актуальность документации |

---

## Связанные файлы

- [01-multi-agent-architecture](01-multi-agent-architecture.md) — evolution steward как часть системы агентов
- [04-observability-and-evals](04-observability-and-evals.md) — eval results как input для эволюции
- [05-knowledge-and-memory](05-knowledge-and-memory.md) — doc-gardening
- [08-security-and-governance](08-security-and-governance.md) — ограничения самомодификации
