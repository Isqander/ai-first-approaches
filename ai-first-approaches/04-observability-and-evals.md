# 04 — Наблюдаемость и оценка (Observability & Evals)

> Как видеть, что делают агенты, и как измерять качество их работы.

---

## 1. Зачем наблюдаемость в AI-системах

В традиционном ПО баг = воспроизводимый. В AI-системе:
- Один и тот же промпт → разные результаты (non-determinism)
- «Баг» часто = плохой контекст, а не плохой код
- Проблемы проявляются статистически, не детерминированно

**Без наблюдаемости** → реактивный loop: «что-то сломалось → чинить вслепую → опять сломалось».
**С наблюдаемостью** → проактивный: «вижу деградацию → понимаю причину → чиню до того, как пользователь заметит».

---

## 2. Три столпа наблюдаемости

### 2.1 Трейсы (Traces)

Каждый запуск агента — trace с уникальным ID. К нему привязано:

```
trace-id: abc123
├── event: user_message (Telegram)
├── event: router_classification (intent: create_task, model: gpt-4o-mini, tokens: 150, latency: 800ms)
├── event: task_created (id: task-456)
├── event: worker_dispatched (provider: claude-code, worktree: wt-789)
├── event: tool_call (tool: search_documents, input: "...", output: "...", latency: 200ms)
├── event: tool_call (tool: write_file, path: "...", latency: 50ms)
├── event: worker_completed (status: success, tokens_total: 15000)
├── event: merge_steward (status: merged, tests: passed)
└── event: user_notified (channel: telegram)
```

**Ценность:** можно проследить любой запрос от Telegram до результата. Понять, где задержка, где ошибка, какой контекст видел агент.

### 2.2 Логирование (Structured Logging)

Формат: **JSONL** (одна JSON-строка = одно событие):

```json
{"ts":"2026-03-14T10:00:00Z","level":"INFO","trace":"abc123","agent":"router","event":"classify","intent":"create_task","confidence":0.95,"tokens":150,"latency_ms":800}
```

Уровни:
- **DEBUG**: детали tool calls, raw LLM responses (для отладки)
- **INFO**: actions taken, decisions made (для мониторинга)
- **WARN**: retries, fallbacks, slow responses (для внимания)
- **ERROR**: failures, crashes, unexpected states (для реакции)

### 2.3 Метрики (Metrics)

| Метрика | Что измеряет | Где |
|---------|-------------|-----|
| **Tokens per task** | Стоимость задачи | MetricsService |
| **Latency** | Время от запроса до ответа | Per-event timestamps |
| **Success rate** | % задач, завершившихся успешно | Task completion events |
| **Fallback rate** | Как часто переключаемся на другой provider | Rate limit events |
| **Limit utilization** | Сколько лимитов потратили vs. доступно | onWatch + MetricsService |
| **Compaction ratio** | Насколько эффективно компактизируем | Compaction events |

---

## 3. Application Legibility (Harness Engineering)

Ключевая идея OpenAI: приложение должно быть **читаемым для агентов**, не только для людей.

### 3.1 UI как инструмент агента

- Chrome DevTools Protocol → агент может «видеть» UI
- Structured error messages → агент понимает, что пошло не так
- Status endpoints → агент проверяет health

### 3.2 Observability Stack

```
Приложение
   ├─ Structured logs (JSONL) → SQLite / Elasticsearch
   ├─ Metrics (tokens, latency, costs) → MetricsService
   ├─ Traces (request→response chain) → Event-sourced JSONL
   └─ Dashboards (Stoneforge smithy-web) → визуализация
```

### 3.3 Remediation Instructions

При ошибках — не просто «Error 500», а инструкция для агента:

```json
{
  "error": "rate_limit_exceeded",
  "provider": "anthropic",
  "retry_after_seconds": 300,
  "remediation": "Switch to codex provider or wait 5 minutes",
  "fallback_chain": ["codex", "opencode"]
}
```

Custom linters тоже выдают remediation:

```
ERROR: Direct import from internal module.
  File: src/stewards/digest.ts:5
  Rule: no-cross-boundary-import
  Fix: Import from public API: `import { DigestService } from '@ai-os/knowledge'`
```

---

## 4. Eval-Driven Development

### 4.1 Что такое Eval

**Eval** (evaluation) — формальная проверка качества AI-компонента:

```
Task (входные данные + ожидаемый результат)
   → Agent/Model (выполняет задачу)
   → Grader (оценивает результат)
   → Outcome (score, pass/fail, transcript)
```

### 4.2 Типы eval'ов

| Тип | Зачем | Пример |
|-----|-------|--------|
| **Capability** | Может ли система это? | «Роутер правильно классифицирует 95% интентов?» |
| **Regression** | Не сломали ли мы что-то? | «После обновления роутер всё ещё работает?» |
| **Quality** | Насколько хорошо? | «Дайджесты информативны и без воды?» |

### 4.3 Graders

| Grader | Когда | Пример |
|--------|-------|--------|
| **Code-based** | Есть точный ответ | `assert router.classify("status") == "command"` |
| **Model-based (LLM-as-a-judge)** | Нет точного ответа, но есть критерии | «Оцени дайджест: 1-5 по информативности» |
| **Human** | Субъективная оценка | Человек ревьюит выборку |

### 4.4 LLM-as-a-Judge

Вторая модель оценивает результат первой:

```
System: You are a quality evaluator for news digests.
Criteria:
- Relevance (1-5): Are the topics relevant to AI and business?
- Conciseness (1-5): No fluff, no repetition?
- Coverage (1-5): Important topics not missed?
- Actionability (1-5): Can the reader act on this info?

Input: [digest text]
Output: { relevance: 4, conciseness: 5, coverage: 3, actionability: 4, feedback: "..." }
```

**Когда использовать:**
- Нет однозначно правильного ответа
- Нужна масштабируемая оценка (100+ примеров)
- Человеческая оценка слишком дорога для каждого запуска

**Когда НЕ использовать:**
- Есть точный expected output (используй code-based)
- Оценка субъективна и зависит от предпочтений (используй human)

### 4.5 Non-determinism: pass@k и pass^k

AI-системы недетерминированы. Одна попытка ≠ надёжная оценка.

| Метрика | Что измеряет | Формула |
|---------|-------------|---------|
| **pass@k** | Хотя бы 1 из k попыток успешна | Optimistic: «может ли?» |
| **pass^k** | Все k попыток успешны | Pessimistic: «надёжно ли?» |

Рекомендация: запускай **минимум 3 попытки** для capability evals. Для regression — 5+.

---

## 5. Roadmap к зрелой системе eval'ов

### Фаза 1: Ручные проверки (MVP)

- Человек ревьюит результаты вручную
- Structured logging для post-mortem
- Базовые code-based тесты для роутера

### Фаза 2: Полуавтоматические evals

- Eval suite для роутера (100+ примеров intent → expected classification)
- LLM-as-a-judge для дайджестов
- Запуск при изменениях в prompts/routing logic

### Фаза 3: Continuous evals

- Автоматический запуск при каждом PR
- Regression detection
- Eval-driven prompt optimization

### Фаза 4: Production monitoring

- Online evals: выборочная оценка production results
- Anomaly detection: отклонения от baseline
- Evolution steward использует eval results для приоритизации улучшений

---

## 6. Eval Frameworks (2026)

| Framework | Тип | Для чего |
|-----------|-----|---------|
| **Promptfoo** | OSS | Eval'ы промптов, сравнение моделей |
| **Langfuse** | OSS | Traces + evals + monitoring |
| **Braintrust** | SaaS | Eval management + LLM-as-a-judge |
| **Harbor** | OSS | Benchmark suite для агентов |
| **Custom (recommended)** | — | Простые скрипты: input → agent → grader → score |

Для персональной системы **custom eval scripts** проще и достаточны. Frameworks — когда масштаб вырастет.

---

## 7. Практические рекомендации

1. **Начинай с логирования** — это фундамент. Без structured logs нет ничего
2. **Каждый agent run = trace** — от trigger до completion
3. **Eval-driven prompts** — изменил промпт → прогони eval suite → сравни с baseline
4. **Measure what matters** — tokens/cost, latency, success rate. Не 50 метрик, а 5 критичных
5. **Post-mortems** — при каждом серьёзном сбое: что случилось, почему, что сделать чтобы не повторилось → Document (category: `post-mortem`)

---

## Связанные файлы

- [03-agent-driven-development](03-agent-driven-development.md) — eval-driven development в контексте AIDD
- [02-context-engineering](02-context-engineering.md) — как плохой контекст влияет на результаты
- [07-self-modification](07-self-modification.md) — evolution steward использует eval results
