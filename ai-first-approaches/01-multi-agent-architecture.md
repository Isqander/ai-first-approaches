# 01 — Мультиагентная архитектура

> Паттерны построения систем из нескольких AI-агентов: роли, оркестрация, координация, масштабирование.

---

## 1. Основные паттерны оркестрации

### 1.1 Planner-Worker-Judge (Cursor)

Трёхролевая модель, проверенная на масштабе (Cursor строил браузер этим подходом):

```
Planner (мощная модель, напр. GPT-5.2 / Claude Opus)
   ├─ Декомпозиция цели на задачи
   ├─ Определение зависимостей
   └─ Распределение по Workers
        │
Workers (N штук параллельно, каждый в изолированной среде)
   ├─ Выполнение конкретной задачи
   └─ Отчёт о результате
        │
Judge (быстрая модель с точными критериями)
   ├─ Валидация: тесты, lint, соответствие спецификации
   └─ accept / reject → Planner получает фидбэк
```

**Ключевые инсайты Cursor:**
- Модель важнее промпта, промпт важнее инфраструктуры
- Простота системы оркестрации > сложные state machines
- Judge — gatekeeper: если тесты не проходят, задача возвращается
- Workers не должны координироваться между собой; координация — задача Planner

### 1.2 Director-Router-Workers (AI OS)

Двухуровневая маршрутизация с эскалацией:

```
Event Handlers (детерминированные, без LLM, <1 сек)
   ├─ /commands → прямая обработка
   ├─ forwarded posts → Document creation
   └─ voice → whisperbot → text
        │
Director-Router (лёгкая модель: gpt-4o-mini)
   ├─ Классификация intent
   ├─ Простые ответы
   └─ Эскалация сложных →
        │
Director-Architect (мощная модель: Claude Code SDK, persistent session)
   ├─ Декомпозиция целей
   ├─ Создание плана с зависимостями
   └─ Dispatch → Workers
```

**Преимущества:** 60-70% запросов обрабатываются без LLM вообще. Лёгкий router отсекает ещё 20%. На мощную модель уходят только реально сложные задачи.

### 1.3 Swarm / Fleet (Background Agents)

Массовый запуск агентов для однотипных задач:

```
Fleet Manager
   ├─ Определяет пул задач (напр. "обновить 50 микросервисов")
   ├─ Создаёт N workers
   └─ Мониторит прогресс
        │
Worker-1 ─── изолированная среда ─── git worktree
Worker-2 ─── изолированная среда ─── git worktree
Worker-N ─── изолированная среда ─── git worktree
        │
Merger (validates + merges results)
```

Актуально для: массовых миграций, security audits, code reviews.

---

## 2. Роли агентов

### 2.1 Типология

| Тип | Lifetime | Инициатор | Пример |
|-----|----------|-----------|--------|
| **Director** | Persistent session | Входящий запрос | Router, Architect |
| **Worker** | Ephemeral (1 задача) | Director / Steward | Код, ресёрч, скрапинг |
| **Steward** | Daemon (cron/event) | Расписание / событие | Merge, digest, monitor |
| **Judge** | Per-evaluation | Worker completion | Валидация результата |

### 2.2 Принцип минимальных привилегий

Каждый агент получает **только те инструменты и права**, которые нужны для его роли:

- **Worker (code)**: git, editor, terminal, tests — но не deployment
- **Worker (research)**: search, scrape, read — но не write/git
- **Steward (merge)**: git merge, CI — но не code editing
- **Director**: planning, task creation, status — но не direct execution

### 2.3 Agent Identity

Каждый агент имеет криптографическую идентичность (Ed25519). Role prompt инжектируется через `CONSTITUTION.md` + role-specific prompt. Агент «знает» кто он, какие у него полномочия, и каковы границы его ответственности.

---

## 3. Competition Mode

Когда качество критично — запуск нескольких agents на одну задачу:

```yaml
playbook: competition
  attempts: 2
  providers: [claude-code, codex]
  evaluate: "judge-prompt + test-results"
  merge: best-result
```

**Когда использовать:**
- Критически важный код (security, financial logic)
- Задача с неоднозначным решением
- Важные архитектурные компоненты

**Когда НЕ использовать:**
- Рутинные задачи (тратит 2x лимитов)
- Задачи с единственным правильным решением
- Мелкие fixes

---

## 4. Координация агентов

### 4.1 Shared State

| Что | Как |
|-----|-----|
| Task queue | Stoneforge Dispatch Daemon (built-in) |
| Results | Event-sourced JSONL |
| Knowledge | Documents Library (shared, read-only для workers) |
| Config | Stoneforge Settings (read-only для workers) |
| Metrics | MetricsService (append-only для workers) |

### 4.2 Изоляция

- Каждый worker в **отдельном git worktree** — нет конфликтов файлов
- Merge через **MergeSteward** с CI-проверкой
- Workers **не коммуницируют между собой** — только через Planner/Director
- Failure одного worker не влияет на других

### 4.3 Dispatch Strategy

```
Dispatch Daemon
   ├─ Получает Task из Quarry
   ├─ Проверяет: есть ли свободные Agent Pools?
   ├─ Проверяет: есть ли доступный provider (не в rate limit)?
   ├─ Создаёт Worker → worktree → role prompt → tools
   └─ Мониторит: timeout, orphan detection, retry policy
```

---

## 5. Fallback и отказоустойчивость

### 5.1 Provider Fallback Chain

```
Claude Code → (rate limit) → Codex → (rate limit) → OpenCode → (rate limit) → pause & notify
```

При rate limit — Dispatch Daemon переключает следующий dispatch на другой provider. Текущий Worker не прерывается.

### 5.2 Orphan Recovery

Dispatch Daemon (cron: `*/10 * * * *`) проверяет workers:
- Завис (нет heartbeat >30 мин) → kill + retry
- Crashed → clean worktree + retry
- Success, но merge failed → re-merge

### 5.3 Escalation

Worker не может решить задачу → возвращает failure → Director-Architect получает контекст ошибки → переформулирует задачу или разбивает на подзадачи.

---

## 6. Multi-model routing

Не каждая задача требует мощной модели:

| Сложность задачи | Рекомендуемая модель | Примеры |
|------------------|---------------------|---------|
| **Тривиальная** | gpt-4o-mini / local | Классификация, routing, форматирование |
| **Средняя** | gpt-4o / Claude Sonnet | Код, ресёрч, анализ |
| **Сложная** | Claude Opus / GPT-5 | Архитектура, планирование, декомпозиция |
| **Критическая** | Competition mode | Security, financial, ядро системы |

**Правило:** начинай с минимально достаточной модели. Если качество не устраивает — эскалируй.

### 6.1 12 правил LLM-проектов (релевантное)

- Маленькая модель часто достаточна — проверяй на eval'ах
- Fine-tuning — когда промпт не справляется и есть данные
- RAG — когда нужен доступ к внешним знаниям
- Tools/agents — когда нужны действия в реальном мире
- Оптимизация: batching, caching, quantization

---

## 7. Паттерны из практики 2026

### 7.1 Self-driving Codebases (OpenAI Harness Engineering)

Репозиторий как автономная система: agents патрулируют, находят проблемы, создают PR. Humans — reviewers и стратеги, не кодеры.

### 7.2 Background Agents

Автономные агенты, работающие по расписанию или по событию без участия человека:
- **Scheduled:** ежедневный дайджест, еженедельный audit
- **Event-driven:** PR opened → auto-review, issue created → auto-triage
- **Fleet:** 50 микросервисов → 50 agents → migration

### 7.3 Layered Agent System (HardFork Conference)

```
Layer 5: People & Organization
Layer 4: Skills & Workflows (playbooks, procedures)
Layer 3: Orchestration (routing, planning, dispatch)
Layer 2: Data & Context (memory, knowledge, RAG)
Layer 1: Infrastructure (compute, network, storage)
── Governance spans all layers ──
```

---

## 8. Антипаттерны

| Антипаттерн | Почему плохо | Что делать |
|-------------|-------------|-----------|
| **God Agent** | Один агент делает всё → context overflow | Разделяй на роли с чётким scope |
| **Chatty Agents** | Агенты общаются между собой → latency, token waste | Координация только через Planner |
| **Shared Mutable State** | Два агента пишут в один файл → conflicts | Worktrees + MergeSteward |
| **Overcomplicated Orchestration** | State machine с 50 состояниями | Простая очередь + retry |
| **Agent Self-Coordination** | Агент сам решает куда эскалировать | Routing — задача Director |

---

## Связанные файлы

- [02-context-engineering](02-context-engineering.md) — как управлять контекстом агентов
- [08-security-and-governance](08-security-and-governance.md) — как ограничить полномочия
- [07-self-modification](07-self-modification.md) — как система эволюционирует
