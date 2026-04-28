# 01. Architecture and Code

Методики проверки того, насколько архитектура и кодовая база пригодны для AI-агента. Для репозитория 50–100 KLOC ключевая проблема обычно не в размере как таковом, а в цене восстановления контекста: агенту нужно быстро понять, где живёт доменная логика, где entrypoints, где side effects, какие импорты разрешены и чем проверить изменение.

## Карта методик

| ID | Методика | Главный вопрос |
|----|----------|----------------|
| A1 | Structural alignment | Совпадают ли архитектурная карта, дерево директорий, импорты и тесты? |
| A2 | Codebase topology map | Есть ли компактная карта подсистем, entrypoints, integrations, storage, jobs? |
| A3 | Execution flows | Описаны ли основные runtime-сценарии от входа до побочных эффектов? |
| A4 | Bounded contexts | Видны ли границы модулей, public API и forbidden imports? |
| A5 | Deep modules | Публичные API компактны и содержательны или поверхностны и шумны? |
| A6 | Honest contracts | Есть ли type hints, docstrings, ошибки, invariants, examples на границах? |
| A7 | Side-effect localization | Побочные эффекты локализованы или размазаны по import-time и globals? |
| A8 | Orchestration vs domain | Оркестрация отделена от доменной логики? |
| A9 | Size/complexity thresholds | Измеряются ли god files, цикломатическая сложность, fan-in/fan-out, циклы? |
| A10 | Architecture as code | Архитектурные правила исполнимы как tests/linters/checks? |
| A11 | Zero-context survival | Можно ли понять модуль без устных пояснений и скрытого tribal knowledge? |
| A12 | Impact analysis | Понятно ли, что затронет изменение? Есть ли symbol/dependency graph, если это действительно нужно? |

## A1. Structural alignment

**Что проверяем.** Архитектурная документация, дерево директорий, namespace, импортные зависимости и тесты должны рассказывать одну и ту же историю.

**Зачем агенту.** Если структура кода отражает архитектуру, агент ищет точку изменения по дереву проекта, а не по догадкам. Если структура случайна, агент тратит основную часть работы на археологию и чаще ломает соседние подсистемы.

**Evidence path.**

1. Найти краткую architecture map: `docs/architecture.md`, `README.md`, `docs/adr/`, `docs/flows/`.
2. Сопоставить названные подсистемы с директориями и пакетами.
3. Проверить реальные импорты: нет ли `domain -> infra`, `core -> web`, `feature_a -> feature_b` без контракта.
4. Проверить, что тесты повторяют структуру кода или явно покрывают ключевые flows.
5. Проверить, что architecture rules enforced в CI или local verify.

**Red flags.**

- В docs есть слои, но в коде только `services/`, `utils/`, `handlers/`.
- Доменная логика размазана между `api/`, `workers/`, `tasks/`, `helpers/`.
- Диаграмма живёт отдельно и не совпадает с импортами.
- Есть пакеты с названиями доменов, но внутри лежат чужие responsibilities.

**Как чинить.** Начинать не с большого рефакторинга, а с `docs/architecture.md` и `docs/architecture-rules.md`: зафиксировать ожидаемые границы, затем ввести 1–3 простых import rules. Рефакторить по одному нарушению, проверяя каждый шаг.

## A2. Codebase topology map

Для 50–100 KLOC нужен компактный файл, который агент может прочитать первым. Он не должен заменять docs, но должен отвечать на вопрос «куда идти дальше?».

Минимальное содержание:

```text
System purpose: 3–5 строк.
Main packages:
- app/api: HTTP entrypoints, no business logic
- app/domain: domain model and use cases
- app/infra: DB, queues, external systems
- app/workers: background jobs, calls domain use cases
Entrypoints:
- web: app/api/main.py
- CLI: app/cli.py
- workers: app/workers/__main__.py
Storage:
- Postgres via app/infra/db
- Redis via app/infra/cache
External integrations:
- payment provider via app/infra/payments
Critical docs:
- docs/flows/order_checkout.md
- docs/adr/0003-payment-boundary.md
Verify:
- just verify
```

Хороший topology map короткий. Детали должны быть ссылками на локальные документы, а не вставками на 20 страниц.

## A3. Execution flows

**Что проверяем.** Есть описания 3–7 критических runtime flows. Например: `create order`, `payment callback`, `background sync`, `data import`, `user auth`, `RAG query`, `scheduled cleanup`.

**Зачем агенту.** Структура директорий показывает где лежит код, а flow показывает как код реально выполняется. Без flows агент видит отдельные файлы, но не понимает последовательность side effects.

**Минимальный формат flow-документа.**

```markdown
# Flow: payment callback

Trigger: POST /payments/callback
Entrypoint: app/api/payments.py:handle_callback
Steps:
1. Validate provider signature: app/infra/payments/signature.py
2. Parse provider event: app/infra/payments/events.py
3. Load order aggregate: app/domain/orders/repository.py
4. Apply payment state transition: app/domain/orders/payment.py
5. Persist order + emit domain event
6. Enqueue receipt job: app/workers/receipts.py
Invariants:
- callback must be idempotent
- domain layer does not import provider SDK
Verify:
- pytest tests/api/test_payment_callback.py
- pytest tests/domain/orders/test_payment_state.py
```

**Red flags.** Flow описан как «система обрабатывает платёж», но не содержит entrypoint, файлов, invariants и verify-команд.

## A4. Bounded contexts and public API

**Что проверяем.** У крупных подсистем есть понятный scope, public API и внутренние детали, в которые нельзя ходить напрямую.

**Evidence.**

- `__init__.py`, `public.py`, `api.py`, `interfaces.py`, `ports.py` или иной явный boundary module.
- Нейминг `internal`, `_private`, `impl`, `adapters` используется осознанно.
- Внешние пакеты импортируют публичный API, а не внутренние классы.
- Запреты enforced через import-linter, tests или CI.

**Рекомендация.** У каждого bounded context должен быть раздел:

```markdown
## Public API
Allowed imports:
- from app.orders import OrderService, OrderId
Forbidden imports:
- app.orders._state
- app.orders.repository_impl
- app.orders.adapters.* from outside app.orders
```

**Red flags.**

- В коде полно импортов вида `from app.orders._internal import ...` из других доменов.
- Изменение одной модели требует правок в 8 пакетах.
- Есть общий `interfaces.py` на весь проект, куда сложены все протоколы.

## A5. Deep modules instead of shallow modules

Deep module — модуль с маленьким понятным интерфейсом и значимой внутренней сложностью. Shallow module — много мелких методов и классов, интерфейс почти такой же сложный, как реализация.

**Что проверяем.**

- Публичные функции/классы решают понятную задачу, а не заставляют вызывающего знать внутренний lifecycle.
- У модуля мало способов использования и явно заданы invariants.
- Наружу не торчат implementation details: session, transaction internals, provider SDK objects, raw DTOs без причины.

**Зачем агенту.** Агент лучше меняет код, когда может доверять публичному контракту, а не читать весь call graph.

**Red flags.**

- `service.prepare()`, `service.validate()`, `service.apply()`, `service.save()`, `service.cleanup()` вызываются вручную из разных мест.
- Чтобы создать объект, нужно знать 12 optional parameters и порядок вызовов.
- Public API возвращает внутренние mutable structures, которые вызывающие модифицируют напрямую.

## A6. Honest contracts

**Что проверяем.** На границах модулей есть честные контракты: type hints, docstrings, exceptions, invariants, examples, stable return types.

**Минимум для Python 50–100 KLOC.**

- Type hints на public API.
- Явные DTO/value objects там, где словари становятся нечитабельными.
- Исключения имеют доменные имена и описанные условия возникновения.
- Docstring не пересказывает код, а фиксирует preconditions/postconditions/invariants.
- Тесты показывают хотя бы happy path и 1–2 important failure cases.

**Плохой контракт.**

```python
def process(data):
    """Processes data."""
```

**Хороший контракт.**

```python
def apply_payment(order: Order, event: PaymentEvent) -> PaymentResult:
    """Apply an idempotent provider payment event to an order.

    Preconditions:
    - event.signature is already verified by infra layer.
    - event.provider_event_id is globally unique.

    Raises:
    - DuplicatePaymentEvent if the event was already applied.
    - InvalidPaymentTransition if the order cannot accept this payment state.
    """
```

## A7. Side-effect localization

**Что проверяем.** IO, network calls, DB writes, queue publishes, filesystem writes, subprocesses and model calls живут в предсказуемых местах.

**Зачем агенту.** Агент должен понимать, где изменение может иметь реальные последствия. Побочные эффекты в import-time коде, глобальных singleton’ах и утилитах ломают reasoning.

**Правила.**

- Не делать network/DB IO при импорте модуля.
- Не создавать скрытые глобальные clients без явного lifecycle.
- Держать side effects в adapters/infra layer.
- Use cases/domain layer возвращают намерения или domain events, а side effects исполняются orchestration/infra layer.
- Конфигурация передаётся явно или через контролируемый settings object.

**Red flags.**

- `import app.settings` запускает чтение `.env`, подключение к базе и создание клиентов.
- `utils.py` отправляет email, пишет в БД и форматирует строки.
- Тесты требуют реальную сеть без явных integration markers.

## A8. Orchestration vs domain logic

**Что проверяем.** Web handlers, CLI commands, workers and scheduled jobs не содержат доменных правил; они парсят вход, вызывают use case, управляют transactions, возвращают response.

**Good shape.**

```text
api/        parse HTTP, auth, response mapping
cli/        parse CLI args, call use cases
workers/    deserialize jobs, call use cases
domain/     entities, rules, policies, use cases
infra/      DB, queues, external APIs
```

**Red flags.**

- HTTP handler на 300 строк содержит бизнес-правила и SQL.
- Worker напрямую меняет доменные состояния без use case.
- CLI, API и worker реализуют один и тот же алгоритм по-разному.

## A9. Size, complexity and structural metrics

Метрики не заменяют ревью, но помогают агенту и человеку видеть очевидные hotspots.

**Alert thresholds, не догма.**

| Метрика | Предупреждение | Сильный red flag |
|---------|----------------|------------------|
| Файл | `>300` строк | `>500` строк без явной причины |
| Функция/метод | `>50` строк | `>100` строк или много nested branches |
| Cyclomatic complexity | `>10` | `>15–20` в core logic |
| Fan-out | модуль импортирует слишком много зон | orchestration/domain/infra смешаны |
| Cycles | есть import cycles | cycles через доменные boundaries |
| Utils/common | много generic helpers | `utils.py` содержит domain/IO/business rules |

**Что проверять.**

- Самые большие файлы и функции.
- Модули с максимальным fan-in/fan-out.
- Импортные циклы.
- Изменяемые hotspots: файлы часто меняются и имеют высокую сложность.
- Доля тестов для hotspots.

## A10. Architecture as code and fitness functions

Архитектурные правила должны быть исполнимыми. Это может быть import-linter, custom pytest, scripts, static analysis, CI gates, generated dependency graph или checks в task runner. Идея fitness functions — дать objective integrity assessment архитектурного свойства [E25][E26].

**Минимальный набор для Python.**

1. Запрет импортов из `infra` в `domain`.
2. Запрет импортов из `api` в `workers` и наоборот, если они независимы.
3. Запрет cross-domain imports без public API.
4. Проверка отсутствия import cycles.
5. Threshold check для больших файлов/functions.

**Пример `.importlinter`.**

```ini
[importlinter]
root_package = app

[importlinter:contract:layers]
name = Layering: api/workers -> domain -> infra interfaces only
# Список сверху вниз: верхние слои могут зависеть от нижних, нижние не могут зависеть от верхних.
type = layers
layers =
    app.api
    app.workers
    app.domain
    app.infra

[importlinter:contract:domain_no_infra_impl]
name = Domain must not import infra implementations
type = forbidden
source_modules = app.domain
forbidden_modules = app.infra
```

Важно: конкретный порядок слоёв зависит от проекта. Не копировать пример без адаптации.

## A11. Zero-context survival

Модуль проходит zero-context survival, если новый агент может открыть его и понять:

- зачем он существует;
- какие входы/выходы у public API;
- какие invariants он защищает;
- где тесты;
- какие соседние модули безопасно менять вместе с ним;
- какие операции опасны.

**Проверка.** Выбрать 5 случайных модулей в core domain и 5 в infra/api. Для каждого дать агенту 5 минут на чтение файла + локального README/docs. Если агент не может описать responsibility и verify-команду, модуль не проходит.

## A12. Impact analysis and repository-level graphs

Для 50–100 KLOC обычно достаточно tree/grep/symbol search + import graph. Symbol/call/dependency graph нужен, когда:

- много cross-file flows;
- изменения часто ломают дальние зависимости;
- language server/IDE navigation недостаточно;
- агент тратит много времени на «где используется?» и «что затронет?»;
- RAG по текстовым chunks даёт шум.

Research по RepoGraph показывает ценность repository-level code graphs для AI software engineering: они дают структуру навигации по репозиторию, а не только file-level context [E23]. Более свежие работы также подчёркивают, что coding agents могут быстро принимать implicit architecture decisions, поэтому структура и review архитектурных последствий становятся важнее [E24].

**Что хранить в Git.**

- Config генерации graph/index.
- Lightweight schema/README по graph.
- Eval queries для impact analysis.

**Что обычно не хранить в Git.**

- Большие generated graph dumps.
- Vector index binary files.
- Raw embeddings.
- Session-specific summaries.

Исключение возможно, если артефакт мал, детерминирован, нужен для offline tooling и явно пересобирается командой.

## Итоговые вопросы для агента

1. Может ли агент по `AGENTS.md` + architecture map найти нужный домен без чтения половины проекта?
2. Совпадает ли «как проект описан» с «как проект импортируется»?
3. Есть ли 3–7 execution flows с entrypoints, invariants и verify?
4. Есть ли public API boundaries и enforced forbidden imports?
5. Локализованы ли side effects?
6. Есть ли objective checks на архитектурный drift?
7. Понятно ли, что затронет изменение, без дорогостоящего ручного поиска?
