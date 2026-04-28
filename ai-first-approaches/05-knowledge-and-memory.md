# 05 — Знания и память (Knowledge & Memory)

> Как AI-системы хранят, находят и используют знания: от AGENTS.md до knowledge graphs.

---

## 1. Репозиторий как система знаний

### 1.1 AGENTS.md — оглавление, не энциклопедия

Ключевой инсайт OpenAI (Harness Engineering): AGENTS.md — это **table of contents**, ~100 строк.

**Что должен содержать:**
- Краткое описание проекта (2-3 предложения)
- Структура директорий со ссылками
- Таблица ключевых документов (priority + зачем)
- Принципы (5-8 штук, не 50)
- Что **не** делать без подтверждения

**Что НЕ должен содержать:**
- Полные API descriptions (→ в docs/)
- Полные архитектурные решения (→ в docs/adr/)
- Инструкции по deployment (→ в docs/ или README)
- Историю изменений (→ в changelog)

```markdown
# AGENTS.md — навигация по проекту
## Что это: [2 предложения]
## Структура: [tree с ссылками]
## Ключевые документы: [таблица: приоритет | файл | зачем]
## Принципы: [5-8 пунктов]
## Не делать без подтверждения: [список]
```

### 1.2 Progressive Disclosure

Агент не должен читать всё сразу. Информация раскрывается **по мере необходимости:**

```
Уровень 0: AGENTS.md (~100 строк) — карта
   → Агент знает, ЧТО есть и ГДЕ

Уровень 1: README.md в каждой директории — обзор
   → Агент знает, КАК устроен модуль

Уровень 2: Конкретные docs/ — детали
   → Агент знает, ЧТО именно делать

Уровень 3: Код + inline docs — реализация
   → Агент может работать с кодом
```

**Аналогия:** файловая система не загружает все файлы в RAM. Загружает directory listing, потом конкретный файл по запросу.

### 1.3 Repository Knowledge as System of Record

Документация в Git — **единственный источник правды**:

| Что | Где | Формат |
|-----|-----|--------|
| Архитектурные решения | `docs/adr/` | ADR (status, context, decision, consequences) |
| Требования | `docs/specs/` | Structured markdown |
| Changelog | `docs/changelog.md` или CHANGELOG.md | По датам или по версиям |
| Знания о домене | `docs/knowledge/` или Documents Library | MD + metadata |
| Operational knowledge | `docs/runbooks/` | Step-by-step procedures |

---

## 2. Documents Library

### 2.1 Архитектура

```
Документ = {
  id: string,
  title: string,
  content: string (markdown),
  category: enum (spec, decision-log, changelog, ...),
  tags: string[],
  version: number,
  created_at: Date,
  updated_at: Date
}
```

### 2.2 Поиск: Hybrid Search

```
Query → FTS5 (keyword match) ─┐
                                ├─ RRF Fusion → ranked results
Query → Embeddings (semantic) ─┘
```

- **FTS5:** находит точные термины, имена, идентификаторы
- **Embeddings (bge-base-en-v1.5):** находит по смыслу, даже если терминология отличается
- **RRF (Reciprocal Rank Fusion):** комбинирует оба списка

### 2.3 Версионирование

Документы версионируются. При обновлении — новая версия, старая сохраняется. Можно:
- Сравнить версии
- Откатить на предыдущую
- Видеть историю изменений

---

## 3. Долгосрочная память

### 3.1 Типы памяти

| Тип | Хранение | Пример |
|-----|---------|--------|
| **Episodic** (что произошло) | Event-sourced JSONL | «Вчера пользователь отклонил идею X» |
| **Semantic** (знания) | Documents Library | «Stoneforge поддерживает playbooks» |
| **Procedural** (как делать) | Skills, Workflows, Runbooks | «Чтобы обновить Stoneforge: npm update + тесты» |

### 3.2 Agent-Directed Retrieval

Агент **сам** решает, когда обратиться к памяти:

```
Agent receives task: "Создай дайджест"
   → Tool call: search_documents(query="digest format preferences", category="decision-log")
   → Gets: "Дайджест должен быть без воды, с ссылками, по категориям"
   → Uses this context for generation
```

Не автоматическая загрузка «всего, что может быть полезно» (context stuffing), а **целенаправленный запрос**.

### 3.3 Memory Update Cycle

```
1. Событие происходит (задача, разговор, решение)
2. Event logged → JSONL (source of truth)
3. Compaction → summary → Document (category: meeting-notes)
4. Если стратегическое решение → Document (category: decision-log)
5. Agent queries memory → gets relevant context
```

---

## 4. Knowledge Graphs для кода

### 4.1 Типы графов

| Граф | Что показывает | Для чего |
|------|---------------|---------|
| **Call Graph** | Кто вызывает кого | Понимание зависимостей, impact analysis |
| **Service Interaction Graph** | Какие сервисы общаются | Архитектура, failure analysis |
| **Message Flow Graph** | Как данные протекают | Debug, optimization |
| **Dependency Graph** | Что от чего зависит | Upgrade planning, security audit |
| **File Tree + Documentation** | Структура проекта | Onboarding, navigation |

### 4.2 Инструменты построения

| Инструмент | Языки | Что строит |
|-----------|-------|-----------|
| **Tree-sitter** | Любой (parsers) | AST → call graph (лёгкий) |
| **Joern** | Java, JS, Python, C | Code Property Graph (тяжёлый, точный) |
| **CodeQL** | Java, JS, Python, Go, Ruby | Query-based analysis |
| **SCIP (Sourcegraph)** | Java, Kotlin, TS, Go | Compiler-accurate symbol index |

### 4.3 Практическое применение для LLM

Граф → MCP tool → агент запрашивает контекст:

```
Agent: tool_call: find_callers("DigestSteward.run")
Result: [
  "DispatchDaemon.dispatchSteward() → DigestSteward.run()",
  "tests/stewards/digest.test.ts → DigestSteward.run()"
]
```

**Когда это нужно:**
- Кодовая база >50K строк
- Множество модулей с неявными зависимостями
- Рефакторинг с analysis of impact

**Когда это overkill:**
- MVP стадия (мало кода для анализа)
- Один разработчик, знающий весь код
- Код хорошо структурирован с явными импортами

### 4.4 SCIP Indexing

Sourcegraph SCIP — compiler-accurate symbol index:

```
scip-java index → .scip file → search API
```

Даёт: «где определён символ», «где используется», «кто наследует». Точнее regex-поиска, но требует инфраструктуры.

---

## 5. Doc-Gardening

### 5.1 Проблема

Документация устаревает. Код меняется, а docs — нет. Агент читает устаревший docs → делает неправильно.

### 5.2 Решение: Doc-Gardening Steward

Периодический (weekly) steward:

```
1. Сканирует docs/ на предмет:
   - Файлы, не обновлявшиеся >30 дней
   - Ссылки на несуществующие файлы
   - Противоречия между документами
   - TODO/FIXME в документах
2. Формирует отчёт → Document (category: changelog)
3. Для очевидных исправлений → Task → Worker → PR
4. Для неочевидных → backlog item
```

### 5.3 Garbage Collection (Harness Engineering)

По аналогии с GC в runtime — периодическая «уборка» репозитория:

- Удаление неиспользуемого кода (dead code)
- Удаление устаревших TODO
- Обновление устаревших комментариев
- Удаление пустых/stub файлов
- Слияние дублирующихся документов

---

## 6. Execution Flow Documentation

### 6.1 Что это

Вместо (или в дополнение к) статической документации — **описание потоков выполнения:**

```markdown
## Flow: Telegram Message → Digest

1. User sends message in Telegram
2. grammy webhook → AI OS Shell
3. Event handler: not a command → Director-Router
4. Router classifies: intent = "request_digest"
5. Router calls: DigestSteward.runNow()
6. DigestSteward:
   a. Reads Inbox (last 24h)
   b. Reads Documents (category: digest, created: last 24h)
   c. Formats markdown
   d. Sends to Telegram
   e. Creates Document (category: digest)
```

### 6.2 Ценность для агентов

Агент, получая задачу «исправь баг в дайджестах», может:
1. Найти Flow Description
2. Понять всю цепочку
3. Определить, где проблема
4. Исправить точечно

---

## 7. Практические рекомендации

1. **AGENTS.md — карта, не руководство.** 100 строк. Ссылки на docs/
2. **Каждый факт — в одном месте** (Single Source of Truth). Дублирование = противоречия
3. **Документация — часть Definition of Done.** Реализовал фичу → обнови docs
4. **Agent-directed retrieval > auto-retrieval.** Агент лучше знает, что ему нужно
5. **Knowledge graphs — инвестиция в будущее.** На MVP — Tree-sitter достаточно. Joern/SCIP — когда кодовая база вырастет
6. **Doc-gardening — гигиена.** Без неё энтропия документации растёт экспоненциально

---

## Связанные файлы

- [02-context-engineering](02-context-engineering.md) — retrieval как часть context pipeline
- [06-codebase-as-prompt](06-codebase-as-prompt.md) — организация кода для AI-читаемости
- [07-self-modification](07-self-modification.md) — doc-gardening как часть self-improvement
