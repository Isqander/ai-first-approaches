# 03 — Agent-Driven Development

> Методологии разработки ПО, где AI-агенты — основные исполнители, а человек — архитектор и стратег.

---

## 1. Фундаментальный сдвиг

**Традиционная разработка:** человек пишет код, AI помогает (autocomplete, copilot).
**Agent-Driven Development:** человек определяет **что** и **почему**, AI определяет **как** и реализует.

```
Человек                          AI-агент
──────                          ────────
Видение, цели                   Декомпозиция
Архитектурные решения            Реализация
Quality bar                     Тестирование
Review & approval               Итерация
```

---

## 2. AIDD — AI-Driven Development Methodology

### 2.1 Принципы

| # | Принцип | Суть |
|---|---------|------|
| 1 | **Developer = Architect** | Человек формулирует требования, принимает решения, оценивает результат |
| 2 | **LLM = Implementer** | AI пишет код, тесты, документацию по спецификации |
| 3 | **Context First** | Качество результата определяется качеством входного контекста |
| 4 | **Documentation as Interface** | Документация — основной канал коммуникации между человеком и агентом |
| 5 | **Iterative Refinement** | Цикл: spec → implement → review → refine |

### 2.2 Проектная документация

Структура документации по AIDD:

```
doc/
├── idea.md            — Первичная идея, проблема, мотивация
├── vision.md          — Видение продукта, целевая аудитория, метрики
├── tech/
│   ├── stack.md       — Стек технологий с обоснованием
│   └── adr/           — Architecture Decision Records
│       ├── 001-*.md
│       └── 002-*.md
└── tasks/
    └── tasklist-<scope>.md  — Задачи по конкретному scope
```

### 2.3 Баланс документации

| Проблема | Как решать |
|---------|-----------|
| Слишком кратко → агент домысливает | Конкретные примеры, edge cases, constraints |
| Слишком подробно → агент теряется | Outline-first: structure, потом детали по запросу |
| Устарело → агент делает неправильно | Актуализация после каждого цикла реализации |
| Дублирование → противоречия | Single Source of Truth: каждый факт — в одном месте |

### 2.4 Уровни абстракции

| Уровень | Документ | Пример |
|---------|---------|--------|
| **Бизнес** | vision.md | «Система должна генерировать дайджесты» |
| **Архитектурный** | adr/ | «Дайджесты создаёт digest-steward через cron» |
| **Технический** | task description | «Реализовать DigestSteward: cron 0 9, source: Inbox+Documents, output: Telegram + Document» |
| **Код** | Inline docs | JSDoc, type signatures |

---

## 3. Spec-Driven Development

### 3.1 Спецификация как исходный код

Вместо «напиши мне X» — формальная спецификация:

```markdown
## Feature: Document Search

### Input
- query: string (user's search query)
- filters: { category?: string, dateRange?: [Date, Date] }

### Behavior
1. Hybrid search: FTS5 + embeddings (RRF Fusion)
2. If category filter → restrict to category
3. Return top-K results sorted by relevance

### Output
- Array of { id, title, snippet, score, category }

### Constraints
- Response time: <500ms for FTS5, <2s with embeddings
- Max results: 20

### Edge Cases
- Empty query → return recent documents
- No results → suggest related categories
```

### 3.2 Contract-Driven Development (GRACE)

Из GRACE methodology — контракт модуля **до** кода:

```
1. Semantic Markup    — описание модуля в structured format
2. Knowledge Graph    — связи с другими модулями
3. Contract           — входы, выходы, инварианты, side effects
4. Implementation     — код, соответствующий контракту
5. Verification       — тесты, проверяющие контракт
```

Контракт включает:
- **Preconditions:** что должно быть истинно до вызова
- **Postconditions:** что гарантированно после вызова
- **Invariants:** что не должно измениться
- **Side effects:** что изменится во внешнем мире

---

## 4. Subagent-Driven Development

### 4.1 Концепция (Superpowers)

Разработка как workflow из composable skills:

```
brainstorm → spec → plan → execute (subagents) → review → merge
```

Каждый этап — отдельный skill. Execute — самый интересный:

### 4.2 Принцип изоляции

Вместо того чтобы один агент писал всё:

```
Main Agent (контекст проекта, план)
   ├─ dispatch subagent: "Реализуй DigestSteward" (с task spec)
   ├─ dispatch subagent: "Реализуй LimitSync" (с task spec)
   └─ dispatch subagent: "Напиши тесты для Router" (с task spec)
```

**Каждый subagent:**
- Получает **только свой** task spec + relevant code
- Работает в **изолированном** git worktree
- Не знает про другие задачи
- Возвращает результат для review

**Зачем:** главный агент не теряет контекст на details of implementation. Subagent работает с чистым context window.

### 4.3 Двухстадийный review (Superpowers)

```
Stage 1: Spec Compliance
   — Соответствует ли результат спецификации?
   — Покрыты ли edge cases?
   — Есть ли тесты?

Stage 2: Code Quality
   — Соответствует ли стилю проекта?
   — Нет ли code smells?
   — Нет ли security issues?
```

---

## 5. Plan Mode

### 5.1 Когда использовать

| Да | Нет |
|----|-----|
| Задача затрагивает >3 файлов | Мелкий fix в 1 файле |
| Несколько валидных подходов | Единственный очевидный подход |
| Архитектурные последствия | Локальное изменение |
| Неясный scope | Чёткая формулировка |

### 5.2 Как работает

```
1. Read widely (greedy reading) — понять контекст
2. Formulate plan — описать подход, alternatives considered
3. Get approval — от человека или Judge-агента
4. Execute — по плану, шаг за шагом
5. Verify — тесты, lint, manual check
```

---

## 6. Greedy Reading

### 6.1 Принцип

Перед любым изменением — **прочитать больше, чем кажется нужным:**

- Все файлы, которые будут затронуты
- Файлы, которые зависят от затрагиваемых
- Тесты для затрагиваемых модулей
- Документацию, если есть

### 6.2 Когда применять

| Задача | Greedy Reading? |
|--------|----------------|
| Дебаг непонятного бага | Да — читай всю цепочку |
| Рефакторинг | Да — нужно знать все зависимости |
| Новая фича в существующем коде | Да — нужно вписаться в паттерны |
| Новый изолированный модуль | Нет — достаточно интерфейсов |
| Typo fix | Нет |

### 6.3 Формализация

Типичная инструкция в system prompt:

> Before making any changes, read all files that will be affected. If you're debugging, trace the full call chain. If you're adding a feature, understand the existing patterns first. Reading is cheaper than fixing broken code.

---

## 7. Execution Plans как артефакты

### 7.1 Plan как документ

Execution plan — это **документ**, а не просто мысли агента:

```markdown
## Plan: Implement DigestSteward

### Goal
Create a Steward that generates daily digest from Inbox + Documents.

### Steps
1. Create `src/stewards/digest-steward.ts` — main logic
2. Add cron trigger in `stewards.config.ts`
3. Implement `formatDigest()` — Telegram-friendly MD
4. Add Documents Library integration for recent items
5. Write tests: mock Inbox, mock Documents, verify output format

### Dependencies
- Documents Library API (existing)
- Telegram send API (existing)
- Cron scheduler (Stoneforge built-in)

### Risks
- Large Inbox → token overflow in summarization
  → Mitigation: chunk + summarize, then merge summaries
```

### 7.2 Зачем

- **Человек может ревьюить план** до начала реализации
- **Агент не теряет направление** в процессе работы
- **Post-mortem:** можно сравнить план с реальностью
- **Reuse:** аналогичные задачи → адаптация плана

---

## 8. Цикл разработки в AI-first проекте

```
1. Человек формулирует цель/проблему
       ↓
2. Director-Architect: декомпозиция → plan → tasks
       ↓
3. Workers (параллельно):
   ├─ Read (greedy reading)
   ├─ Implement (по spec)
   ├─ Test (autonomous feedback loop)
   └─ Report result
       ↓
4. Judge / Review:
   ├─ Tests pass?
   ├─ Spec compliance?
   └─ Code quality?
       ↓
5. MergeSteward: merge + CI
       ↓
6. Docs update (если нужно)
       ↓
7. Notify human (Telegram)
```

---

## 9. Антипаттерны

| Антипаттерн | Проблема | Что делать |
|-------------|---------|-----------|
| **Vibe Coding** | «Напиши мне что-нибудь» → random quality | Всегда spec, хотя бы минимальный |
| **One-Shot** | Одна попытка без итерации → ошибки | Iterative: implement → test → refine |
| **God Context** | Весь проект в промпте → lost in the middle | Scoped context per task |
| **No Verification** | Агент написал → merge → broken | Autonomous feedback loop: тест до merge |
| **Manual Glue** | Человек собирает результаты вручную | MergeSteward + CI + auto-notify |

---

## Связанные файлы

- [01-multi-agent-architecture](01-multi-agent-architecture.md) — роли Planner/Worker/Judge
- [02-context-engineering](02-context-engineering.md) — scoped context для subagents
- [04-observability-and-evals](04-observability-and-evals.md) — eval-driven development
- [06-codebase-as-prompt](06-codebase-as-prompt.md) — как код должен быть устроен для AI
