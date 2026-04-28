# 06 — Кодовая база как промпт (Codebase as Prompt)

> Как писать и организовывать код так, чтобы AI-агентам было легко его читать, понимать и модифицировать.

---

## 1. Центральная идея

**Архитектура — это промпт.** Структура кода определяет, насколько хорошо AI-агент поймёт систему и сможет вносить изменения. Хорошая архитектура для AI = хорошая архитектура вообще, но с акцентом на **явность** и **изолированность**.

---

## 2. Sinks, Not Pipes

### 2.1 Принцип

**Pipe** (труба) — компонент, через который данные «протекают» с побочными эффектами вниз по цепочке. Изменение pipe может сломать что угодно downstream.

**Sink** (сток) — компонент, который **полностью содержит** свои эффекты. Изменение sink не влияет на другие компоненты.

```
❌ Pipe: OrderService → PaymentService → NotificationService → AnalyticsService
   (изменение OrderService может сломать аналитику)

✅ Sink: OrderService {
     - validates order (contained)
     - processes payment (contained)
     - emits event: OrderCompleted (interface)
   }
   NotificationService listens to OrderCompleted (decoupled)
   AnalyticsService listens to OrderCompleted (decoupled)
```

### 2.2 Почему это важно для AI

- AI-агент видит **один модуль** за раз (ограниченный context window)
- Если модуль = sink → агент может безопасно изменить его, не зная всей системы
- Если модуль = pipe → агент должен понимать всю цепочку → больше контекста → больше ошибок

### 2.3 Правила для sinks

| Правило | Объяснение |
|---------|-----------|
| **Contained effects** | Все побочные эффекты — внутри модуля |
| **Explicit interfaces** | Входы и выходы — через типизированные контракты |
| **Event-based communication** | Между модулями — через события, не прямые вызовы |
| **No hidden state** | Модуль не зависит от скрытого глобального состояния |

---

## 3. Deep Modules

### 3.1 Что это

**Deep module** = маленький интерфейс + богатая реализация.
**Shallow module** = большой интерфейс + тривиальная реализация.

```
✅ Deep: search(query: string, filters?: SearchFilters): SearchResult[]
   Внутри: FTS5, embeddings, RRF fusion, ranking, caching, error handling

❌ Shallow: initSearch(), setQuery(), addFilter(), execute(), getResults(), cleanup()
   Внутри: каждый метод делает одну тривиальную операцию
```

### 3.2 Для AI-агента

- Deep module = **одна точка входа** → агент знает, что вызвать
- Shallow module = **10 методов** → агент должен понять порядок вызовов, lifecycle
- Deep module прячет complexity → агент работает с **abstraction**, не с деталями

### 3.3 Honest Interfaces

Интерфейс должен **честно** описывать, что модуль делает:

```typescript
// ✅ Honest: название и типы говорят всё
async function searchDocuments(
  query: string,
  filters?: { category?: string; dateRange?: [Date, Date] }
): Promise<Array<{ id: string; title: string; snippet: string; score: number }>>

// ❌ Dishonest: непонятно что возвращает, какие side effects
async function process(input: any): Promise<any>
```

---

## 4. Файловая структура = архитектура

### 4.1 Принцип

Файловая структура проекта должна **зеркалить** концептуальную архитектуру:

```
src/
├── interfaces/          — как система общается с внешним миром
│   ├── telegram/        — Telegram bot + userbot
│   ├── frontend/        — React UI
│   └── mcp-gateway/     — MCP proxy
├── directors/           — маршрутизация и планирование
│   ├── router.ts        — Director-Router
│   └── architect.ts     — Director-Architect
├── workers/             — выполнение задач
│   ├── code-worker.ts
│   └── research-worker.ts
├── stewards/            — фоновая автоматизация
│   ├── digest.ts
│   ├── monitor.ts
│   └── evolution.ts
├── knowledge/           — хранилище и поиск
│   ├── documents.ts
│   ├── search.ts
│   └── compaction.ts
├── security/            — безопасность
│   ├── approval.ts
│   └── constitution.ts
└── shared/              — общие утилиты
    ├── types.ts
    ├── events.ts
    └── config.ts
```

**Зачем:** агент, увидев `src/stewards/digest.ts`, мгновенно понимает: это Steward, занимающийся дайджестами. Не нужно читать README, чтобы найти нужный файл.

### 4.2 Bounded Contexts

Каждая директория — **bounded context** с чётким scope:

- `knowledge/` не зависит от `telegram/`
- `stewards/` использует `knowledge/` через public API, не через internal imports
- `shared/` — минимум: types, events, config

```
✅ import { searchDocuments } from '@ai-os/knowledge'
❌ import { _internalFTS5Query } from '../knowledge/internal/fts5'
```

### 4.3 Enforce через Linters

Не полагайся на дисциплину — enforce правила инструментами:

```javascript
// eslint rule: no-cross-boundary-import
// Запрещает импорт из internal/ модулей другого bounded context
{
  "rules": {
    "import/no-restricted-paths": [{
      "zones": [{
        "target": "./src/stewards",
        "from": "./src/knowledge/internal"
      }]
    }]
  }
}
```

Custom linters с **remediation instructions** — агент не только видит ошибку, но и знает, как исправить.

---

## 5. Скрипты > Описания в промптах

### 5.1 Принцип

Для повторяемых операций — **скрипты**, а не инструкции в промптах:

```
❌ "Чтобы запустить тесты, перейди в директорию src/, выполни npm test,
    проверь exit code, если не 0 — посмотри output..."

✅ npm run test:all   ← один скрипт, детерминированный результат
```

### 5.2 Примеры скриптов для агентов

```json
{
  "scripts": {
    "test:all": "vitest run",
    "test:watch": "vitest",
    "lint": "eslint src/ --fix",
    "lint:check": "eslint src/",
    "typecheck": "tsc --noEmit",
    "verify": "npm run lint:check && npm run typecheck && npm run test:all",
    "docs:check": "node scripts/check-docs-links.js",
    "update:examples": "powershell scripts/update-examples.ps1"
  }
}
```

**`verify`** — одна команда, которая проверяет всё. Агент запускает после каждого изменения.

---

## 6. Autonomous Feedback Loop

### 6.1 Принцип

Агент должен уметь **проверить результат своей работы** до того, как отправит его на merge:

```
1. Write code
2. Run tests → pass?
3. Run linter → clean?
4. Run typecheck → no errors?
5. If all pass → submit for merge
6. If fail → analyze error → fix → goto 2
```

### 6.2 DEBUG.md

Документ `DEBUG.md` для каждого runtime/сервиса — verified runbook:

```markdown
## DEBUG.md — AI OS Shell

### How to verify
- `npm run verify` — lint + typecheck + tests
- `npm run test:integration` — integration tests (requires running Stoneforge)

### Common errors
| Error | Cause | Fix |
|-------|-------|-----|
| `ECONNREFUSED :3457` | Stoneforge not running | `docker-compose up stoneforge` |
| `Rate limit` | Provider exhausted | Check onWatch, switch provider |

### Logs
- Structured logs: `logs/ai-os-shell.jsonl`
- Stoneforge logs: `docker-compose logs stoneforge`
```

### 6.3 Принцип «Verify before Document»

Не документировать то, что не проверено:
1. Обнаружил debug/test метод
2. **Проверь** — работает ли он
3. Только потом — в DEBUG.md

---

## 7. Layered Domain Architecture

### 7.1 Слои

```
Interface Layer (Telegram, Frontend, MCP)
       ↓
Application Layer (Directors, Workers, Stewards)
       ↓
Domain Layer (Knowledge, Security, Limits)
       ↓
Infrastructure Layer (Events, SQLite, Git, External APIs)
```

### 7.2 Правила зависимостей

- Каждый слой зависит **только от слоя ниже**
- Interface не импортирует из Infrastructure напрямую
- Domain не знает про Telegram или React

### 7.3 Enforce

```
// Architecture tests (vitest)
test('domain layer does not import from interface layer', () => {
  const domainFiles = glob.sync('src/knowledge/**/*.ts')
  for (const file of domainFiles) {
    const content = readFileSync(file, 'utf-8')
    expect(content).not.toMatch(/from ['"].*interfaces/)
  }
})
```

---

## 8. Code Conventions для AI-читаемости

| Практика | Зачем |
|---------|-------|
| **Типизация (TypeScript strict)** | AI видит типы → понимает контракт без чтения реализации |
| **Именование: полные слова** | `searchDocuments` > `srchDocs`. AI лучше работает с natural language |
| **1 файл = 1 модуль = 1 ответственность** | Агент работает с одним файлом за раз |
| **Export only public API** | Агент видит только то, что нужно; internal — спрятано |
| **Consistent patterns** | Все Stewards имеют одинаковую структуру → агент выучивает паттерн |
| **Тесты рядом с кодом** | `digest.ts` + `digest.test.ts` в одной директории |

---

## 9. Антипаттерны

| Антипаттерн | Проблема для AI | Что делать |
|-------------|----------------|-----------|
| **God File** (>500 строк) | Не влезает в context window целиком | Split by responsibility |
| **Implicit Dependencies** | Агент не видит зависимость → ломает | Explicit imports + DI |
| **Magic Strings** | `if (type === "dgst")` — агент не знает что это | Enums + constants |
| **Circular Imports** | Непонятно, где начало → confusion | Layered architecture |
| **Commented-out Code** | Агент не знает: это архив или мусор? | Delete it. Git помнит |
| **Config in Code** | Агент не может изменить behavior без code change | Config files + env vars |

---

## Связанные файлы

- [05-knowledge-and-memory](05-knowledge-and-memory.md) — как документация влияет на AI-понимание
- [03-agent-driven-development](03-agent-driven-development.md) — autonomous feedback loop в процессе разработки
- [08-security-and-governance](08-security-and-governance.md) — bounded contexts как security boundary
