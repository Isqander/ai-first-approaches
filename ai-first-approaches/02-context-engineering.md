# 02 — Context Engineering

> Управление контекстом AI-агентов как инженерная дисциплина: что агент видит, когда и в каком объёме.

---

## 1. Context Engineering — определение

**Context** — это не просто промпт. Это **скомпилированное представление** всей информации, доступной агенту в момент принятия решения. Включает:

- System prompt (role, constitution, tools description)
- Session history (текущий диалог / task context)
- Retrieved knowledge (из Documents Library, RAG)
- Tool outputs (результаты вызовов инструментов)
- Artifacts (большие объекты: файлы, схемы, данные)

**Качество результата = качество входного контекста** (Context First principle из AIDD).

---

## 2. Tiered Context Model (Google ADK)

Модель из Google Agent Development Kit — четыре уровня контекста:

```
┌─────────────────────────────────────────────┐
│   Working Context (рабочая память)          │
│   ← то, что в context window прямо сейчас   │
├─────────────────────────────────────────────┤
│   Session Context                            │
│   ← история текущей сессии/задачи            │
├─────────────────────────────────────────────┤
│   Memory (долгосрочная память)               │
│   ← Facts, preferences, learned patterns     │
├─────────────────────────────────────────────┤
│   Artifacts (большие объекты)                │
│   ← Файлы, схемы, datasets                  │
└─────────────────────────────────────────────┘
```

### 2.1 Working Context

- То, что реально в context window модели
- **Budget:** ~100K-200K токенов (зависит от модели)
- Включает: system prompt + последние N сообщений + relevant retrievals
- **Критично:** не засорять — каждый лишний токен снижает качество (lost in the middle)

### 2.2 Session Context

- Полная история текущей задачи / разговора
- Хранится в event-sourced JSONL (source of truth)
- Подвергается **компактизации** перед загрузкой в Working Context
- Для длинных сессий: только summary + последние N сообщений

### 2.3 Memory

- Долгосрочные факты: предпочтения пользователя, принятые решения, паттерны
- Хранится в Documents Library (категория `reference`, `decision-log`)
- **Agent-directed retrieval:** агент сам решает, что достать из памяти (через tool calls)
- Обновляется через `meeting-notes`, `post-mortem`

### 2.4 Artifacts

- Большие объекты, которые не влезают в context window целиком
- Хранятся отдельно (файлы в Git, записи в Documents Library)
- Доступ через tools: `read_file`, `search_documents`
- **Reference, не inline:** агент получает ссылку, а не полный текст

---

## 3. Context Transformations

Контекст не передаётся as-is — он трансформируется:

### 3.1 Компактизация (Compaction)

Сжатие session history для освобождения context window:

| Возраст | Объём | Действие |
|---------|-------|---------|
| <24ч | Мало | Без компактизации |
| <24ч | Много | Компактизация самых старых/больших кусков |
| >24ч | Любой | Лёгкая компактизация |
| >7 дней | Любой | Средняя компактизация |
| >30 дней | Любой | Сильная компактизация |

**Приоритеты компактизации:**
1. **Быстро:** длинные вставки (код, логи, JSON) → краткое описание
2. **Средне:** цитаты, пересланные посты → ключевые факты
3. **Медленно:** слова пользователя, ключевые выводы
4. **Никогда:** стратегические и архитектурные решения → в Documents Library

### 3.2 Retrieval (поиск релевантного)

```
Query → Hybrid Search (FTS5 + embeddings, RRF Fusion) → Top-K → Working Context
```

- **Semantic search** (embeddings) — находит по смыслу
- **Full-text search** (FTS5) — находит по точным терминам
- **RRF Fusion** — комбинирует оба подхода

### 3.3 Scoped Handoffs (мультиагентный контекст)

При передаче задачи от Director к Worker — не весь контекст, а **scoped subset:**

```
Director Context (полный)
   └─ Handoff to Worker:
        ├─ Task description (что делать)
        ├─ Relevant context (только то, что нужно для задачи)
        ├─ Constraints (ограничения, quality bar)
        └─ Expected output format
```

**Не передаётся:** история переписки с пользователем, метрики других задач, irrelevant documents.

---

## 4. Context Window Management

### 4.1 Проблема «Lost in the Middle»

Модели плохо обращают внимание на информацию в середине длинного контекста. Следствия:

- **Самое важное — в начале и в конце** контекста
- System prompt — в начале
- Текущая задача — в конце
- Retrieval results — непосредственно перед задачей

### 4.2 Принцип прогрессивного раскрытия (Progressive Disclosure)

Не давать агенту всю информацию сразу. Давать **карту** (map), а не **руководство** (manual):

```
AGENTS.md → ~100 строк, table of contents, ссылки на docs/
   └─ Агент идёт в docs/ только когда конкретная информация нужна
```

Аналогия: IDE не загружает весь проект в RAM — подгружает файлы по запросу.

### 4.3 Context Caching

Для частых паттернов (role prompts, CONSTITUTION, tool descriptions) — кэширование на уровне API:

- Anthropic: prompt caching (prefix caching)
- OpenAI: predicted outputs / prompt caching
- Экономия: до 90% стоимости для повторяющихся префиксов

### 4.4 Стратегия Greedy Reading

Для сложных задач — «жадное чтение» перед действием:

1. Прочитать все релевантные файлы
2. Понять контекст целиком
3. Только потом писать код

Дороже по токенам, но значительно снижает ошибки. Особенно полезно:
- Дебаг (нужно видеть всю цепочку)
- Рефакторинг (нужно понимать зависимости)
- Новый код в существующей кодовой базе

---

## 5. Context для разных ролей

| Роль | Working Context содержит | Не содержит |
|------|------------------------|------------|
| **Director-Router** | Intent classification examples, available commands, user message | Полную историю, код проекта |
| **Director-Architect** | Inbox, worker statuses, knowledge base summary, metrics, limits | Детали реализации workers |
| **Worker (code)** | Task spec, relevant files, test expectations, CONSTITUTION | Другие задачи, бизнес-контекст |
| **Worker (research)** | Research question, search constraints, quality criteria | Код проекта, другие задачи |
| **Steward** | Trigger context, relevant data, action templates | Переписка с пользователем |

---

## 6. Антипаттерны контекста

| Антипаттерн | Проблема | Решение |
|------------|---------|--------|
| **Context Stuffing** | Всё в prompt → lost in the middle | Progressive disclosure + retrieval |
| **Stale Context** | Устаревшая информация → wrong decisions | Актуализация docs, компактизация |
| **Context Leakage** | Worker видит sensitive data другого worker | Scoped handoffs |
| **No Context** | Агент работает вслепую → low quality | Context First: сначала контекст, потом задача |
| **Duplicate Context** | Одна информация в 3 местах → contradictions | Single Source of Truth |

---

## 7. Практические рекомендации

1. **Инвестируй в качество контекста больше, чем в промпты** — хороший контекст с простым промптом лучше, чем плохой контекст с продвинутым промптом
2. **Separate storage from presentation** — хранение (JSONL, Documents) и представление (Working Context) — разные слои с трансформацией между ними
3. **Агент сам управляет своим контекстом** — через tool calls для retrieval, а не через автоматическую загрузку всего
4. **Компактизация — это архитектурный компонент**, а не afterthought. Проектируй с самого начала
5. **Измеряй** — сколько токенов уходит на context vs на generation. Оптимизируй context budget

---

## Связанные файлы

- [01-multi-agent-architecture](01-multi-agent-architecture.md) — роли и scoped handoffs
- [05-knowledge-and-memory](05-knowledge-and-memory.md) — долгосрочное хранение знаний
- [06-codebase-as-prompt](06-codebase-as-prompt.md) — как структура кода влияет на context quality
