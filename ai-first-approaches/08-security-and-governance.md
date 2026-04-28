# 08 — Безопасность и Governance

> Как контролировать AI-агентов: trust levels, sandboxing, HITL, policy enforcement.

---

## 1. Фундаментальная проблема

AI-агенты действуют **автономно**. Без контроля агент может:
- Удалить данные
- Отправить credentials в открытый канал
- Push в main без review
- Потратить все лимиты на одну задачу
- Изменить свои собственные ограничения

**Governance** — система правил и механизмов, обеспечивающих безопасное поведение агентов.

---

## 2. Human-in-the-Loop (HITL)

### 2.1 Принцип

Опасные и необратимые действия требуют **подтверждения человека**:

```
Agent wants to perform action
   ├─ Risk assessment: LOW / MEDIUM / HIGH / CRITICAL
   ├─ LOW → auto-approve
   ├─ MEDIUM → execute + notify
   ├─ HIGH → request approval → wait → execute if approved
   └─ CRITICAL → request approval + backup → wait → execute if approved
```

### 2.2 Risk Classification

| Риск | Примеры | Обработка |
|------|---------|-----------|
| **LOW** | Чтение файлов, поиск, создание документов, retrieval | Автоматически |
| **MEDIUM** | Создание/изменение skills, workflows, конфигов, lint fix | Worker → merge → notify |
| **HIGH** | Изменение кода AI OS Shell, prompts, CONSTITUTION | ApprovalService → Telegram → approval |
| **CRITICAL** | Удаление данных, push в main, деплой, credentials, self-modification security rules | Двойное подтверждение + backup + audit |

### 2.3 ApprovalService (Stoneforge)

```
Worker запрашивает опасное действие
   → ApprovalService перехватывает
   → Формирует описание: что, зачем, risk level, diff
   → Telegram notification → кнопки [Approve] [Reject] [Details]
   → Ожидание (timeout: configurable, default 24h)
   → Если approved → execute
   → Если rejected → abort + log reason
   → Если timeout → abort + notify
```

---

## 3. Sandboxing

### 3.1 Git Worktrees (агентная изоляция)

Каждый Worker работает в **изолированном git worktree**:

```
main branch (production state)
├── worktree/wt-001 (Worker A: task X)
├── worktree/wt-002 (Worker B: task Y)
└── worktree/wt-003 (Worker C: task Z)
```

**Гарантии:**
- Workers не могут изменить main напрямую
- Workers не видят изменения друг друга
- Merge — только через MergeSteward с CI
- При crash — worktree удаляется, main не затронут

### 3.2 Docker Network Isolation

```yaml
services:
  stoneforge:
    networks: [internal, external]
  ai-os-shell:
    networks: [internal, external]
  searxng:
    networks: [internal]      # нет доступа к internal APIs
  mcp-servers:
    networks: [internal]      # изолированы от интернета
```

Принцип: каждый сервис получает **минимально необходимый** network access.

### 3.3 Per-MCP Sandboxing

MCP серверы могут быть ненадёжными (third-party). Изоляция:

| Метод | Когда |
|-------|-------|
| Отдельный Docker container | Для ненадёжных MCP серверов |
| rlimit (memory, CPU) | Для всех MCP серверов |
| Timeout | Для каждого tool call |
| Output size limit | Предотвращение context stuffing |

---

## 4. Принцип наименьших привилегий

### 4.1 Role Definitions

Каждая роль получает **только** необходимые инструменты:

```yaml
role: code-worker
  tools:
    - file_read
    - file_write
    - terminal (restricted to worktree)
    - search_documents
    - run_tests
  forbidden:
    - git_push
    - deploy
    - credential_access
    - modify_constitution

role: research-worker
  tools:
    - search_web
    - scrape_url
    - search_documents
    - create_document
  forbidden:
    - file_write (code files)
    - terminal
    - git operations
```

### 4.2 PreToolUse Hooks

Перехват tool calls **до выполнения** (Claude Code SDK):

```typescript
preToolUse(toolCall) {
  if (toolCall.tool === 'bash' && toolCall.args.includes('rm -rf')) {
    return { action: 'block', reason: 'Destructive command blocked' }
  }
  if (toolCall.tool === 'file_write' && toolCall.path.includes('.env')) {
    return { action: 'require_approval', reason: 'Writing to .env requires approval' }
  }
  return { action: 'allow' }
}
```

### 4.3 Whitelist vs Blacklist

| Подход | Когда | Пример |
|--------|-------|--------|
| **Whitelist** (default deny) | Для критических ролей | Worker может ТОЛЬКО: read, write in worktree, test |
| **Blacklist** (default allow) | Для flexible ролей | Director может всё КРОМЕ: push, deploy, delete |

**Рекомендация:** whitelist для Workers, blacklist для Directors.

---

## 5. CONSTITUTION.md

### 5.1 Назначение

Единый документ, определяющий **идентичность и правила** системы. Инжектируется во все role prompts:

```markdown
# CONSTITUTION.md

## Identity
You are AI OS — a personal AI operating system for Alexander.

## Core Rules
1. NEVER delete data without backup
2. NEVER push to main without review
3. NEVER expose credentials in any channel
4. NEVER modify CONSTITUTION without explicit human approval
5. When in doubt — ASK, don't act

## Risk Assessment
Before any action, assess risk:
- LOW: proceed automatically
- MEDIUM: proceed and notify
- HIGH: request approval first
- CRITICAL: request approval + create backup first

## Behavioral Constraints
- Prefer reversible actions over irreversible
- Prefer less expensive models when quality is sufficient
- Log every significant action
- If you don't know — say so, don't hallucinate
```

### 5.2 Свойства CONSTITUTION

- **Immutable without approval** — изменение = CRITICAL risk
- **Versioned** — каждое изменение = new version + git commit
- **Injected** — автоматически добавляется в каждый role prompt
- **Observable** — UI для просмотра текущей версии

---

## 6. Policy Enforcement

### 6.1 Infrastructure > Prompts

Не полагайся на промпты для безопасности:

```
❌ "Please don't push to main" (агент может проигнорировать)
✅ Git hook, блокирующий push to main без CI (невозможно обойти)
```

| Уровень | Метод | Надёжность |
|---------|-------|-----------|
| **Infrastructure** | Git hooks, Docker network, rlimit | Невозможно обойти |
| **Code** | PreToolUse hooks, ApprovalService | Сложно обойти |
| **Configuration** | Role definitions, whitelist/blacklist | Средне |
| **Prompt** | CONSTITUTION, role prompts | Легко обойти (prompt injection) |

**Правило:** security-critical ограничения — на уровне infrastructure. Prompt — дополнительный слой, не основной.

### 6.2 CI как Security Gate

MergeSteward запускает CI перед каждым merge:

```yaml
testCommand: "npm run lint:check && npm run typecheck && npm run test:all"
```

Если CI не проходит → merge отклонён → Worker получает ошибку → исправляет → retry.

---

## 7. Credential Management

### 7.1 API Keys

```yaml
# docker-compose.yml
services:
  stoneforge:
    env_file: .env
    # .env содержит: ANTHROPIC_API_KEY, OPENAI_API_KEY, etc.
    # .env НЕ коммитится в git
```

### 7.2 CLI Agent Auth

```yaml
volumes:
  - ~/.claude:/home/agent/.claude:ro      # Claude Code auth (read-only)
  - ~/.codex:/home/agent/.codex:ro        # Codex auth (read-only)
```

Read-only mount — агент может **использовать** auth, но не может **изменить** или **экспортировать**.

### 7.3 Telegram Tokens

```
BOT_TOKEN — в .env
USERBOT_SESSION — encrypted, в volume
```

---

## 8. Audit Trail

### 8.1 Event-Sourced Logging

Каждое действие = event в JSONL:

```json
{"ts":"...","agent":"worker-001","action":"file_write","path":"src/foo.ts","risk":"LOW","approved":"auto"}
{"ts":"...","agent":"worker-001","action":"git_push","risk":"HIGH","approved":"user:alexander","approval_id":"ap-123"}
{"ts":"...","agent":"evolution-steward","action":"modify_prompt","risk":"MEDIUM","approved":"auto","diff":"..."}
```

### 8.2 Что логировать

| Событие | Зачем |
|---------|-------|
| Каждый tool call | Трассировка действий |
| Каждое решение о risk level | Аудит классификации |
| Каждый approval/rejection | История решений |
| Каждый rate limit | Понимание resource usage |
| Каждый merge | Что было влито, кем |

### 8.3 Non-repudiation

Ed25519 identity → каждое действие **подписано** агентом. Невозможно подделать или отрицать авторство.

---

## 9. Trust Model

### 9.1 Trust Levels

```
Human (Alexander)       ← максимальный trust
Director-Architect      ← высокий (persistent session, reviews)
Director-Router         ← средний (classification only)
Stewards (built-in)     ← средний (predefined scope)
Workers (code)          ← низкий (ephemeral, sandboxed)
Workers (research)      ← низкий (read-only)
External MCP servers    ← минимальный (sandboxed, output validated)
```

### 9.2 Trust ↔ Capabilities

Чем ниже trust → тем меньше capabilities:
- External MCP: только structured output, ограниченный размер, timeout
- Worker: только worktree, только разрешённые tools
- Director: доступ к planning, но не к execution
- Human: неограниченный (owner)

---

## 10. Threat Model

| Угроза | Вектор | Mitigation |
|--------|--------|-----------|
| **Prompt injection** | Malicious content в Telegram/web | Input sanitization + CONSTITUTION overrides |
| **Tool misuse** | Агент вызывает rm -rf | PreToolUse hooks + whitelist |
| **Credential leak** | Агент выводит API key в лог | Output filtering + secret scanning |
| **Resource exhaustion** | Агент тратит все лимиты | Agent Pools + rate limiting |
| **Self-modification escape** | Агент меняет свои restrictions | Infrastructure enforcement + CONSTITUTION immutability |
| **Supply chain** | Malicious MCP server | Sandboxed MCP + output validation |

---

## 11. Практические рекомендации

1. **Defense in depth** — несколько слоёв: infrastructure + code + prompt
2. **Fail closed** — при неизвестном risk level → HIGH (требовать approval)
3. **Принцип наименьших привилегий** — каждый агент получает минимум необходимого
4. **Audit everything** — event-sourced JSONL для полной трассировки
5. **Test security** — periodic security audit steward
6. **CONSTITUTION — единственная истина** — не дублировать security rules в разных местах
7. **Infrastructure > prompts** для критических ограничений

---

## Связанные файлы

- [01-multi-agent-architecture](01-multi-agent-architecture.md) — роли и их полномочия
- [07-self-modification](07-self-modification.md) — ограничения самомодификации
- [04-observability-and-evals](04-observability-and-evals.md) — audit trail и мониторинг
