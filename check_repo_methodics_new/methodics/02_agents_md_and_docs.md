# 02. AGENTS.md, Agent Instructions and Documentation

Методики проверки слоя инструкций и документации. Цель — чтобы AI-агент получал стабильный, короткий и проверяемый контекст, а не угадывал правила проекта из случайных файлов.

## Карта методик

| ID | Методика | Главный вопрос |
|----|----------|----------------|
| D1 | `AGENTS.md` as navigation | Это карта проекта или энциклопедия/манифест? |
| D2 | Instruction hierarchy | Есть ли root + path-scoped instructions без конфликтов? |
| D3 | Actionable rules | Инструкции содержат команды, границы, стиль, запреты, expected output? |
| D4 | Progressive disclosure | Документы раскрываются слоями, а не грузятся целиком? |
| D5 | Docs as interface | Документация помогает выполнять работу? |
| D6 | Single source of truth | У каждого ключевого факта есть владелец и основной файл? |
| D7 | ADR/spec/flow layer | Решения, требования и runtime flows версионируются? |
| D8 | Doc-gardening | Есть ли процесс/автоматизация против stale docs? |
| D9 | Human vs agent docs | README для людей и agent docs не конфликтуют? |
| D10 | Conflict resolution | Ясно ли, что делать при конфликте инструкций? |

## D1. `AGENTS.md` as navigation, not encyclopedia

`AGENTS.md` — предсказуемое место для инструкций coding agents, дополняющее human-focused README: build steps, tests, conventions и project-specific guidance [E1]. GitHub docs также поддерживают несколько `AGENTS.md` в репозитории: ближайший в дереве может иметь precedence для соответствующей зоны [E2].

**Что проверяем.** Root `AGENTS.md` должен за 2–5 минут дать агенту маршрут работы:

- что это за проект;
- какие директории важны;
- что читать первым;
- как установить зависимости;
- как запустить быстрые и полные проверки;
- какие архитектурные правила нельзя нарушать;
- какие файлы/операции требуют осторожности;
- как должен выглядеть итог работы.

**Что не должно быть в root `AGENTS.md`.**

- Полная архитектурная документация на десятки страниц.
- Длинные списки всех модулей и классов.
- Raw prompts для конкретной модели.
- Случайные chat-derived notes.
- Большие troubleshooting разделы, если их можно вынести в `DEBUG.md` или runbook.

**Хороший размер.** Для 50–100 KLOC обычно полезен root файл около 80–200 строк. Если больше — скорее всего, нужен progressive disclosure.

## D2. Instruction hierarchy

Для крупного проекта нужен layered instruction system:

```text
AGENTS.md                         # root, canonical map
.github/copilot-instructions.md    # optional bridge, if tool needs it
.github/instructions/*.md          # path-specific Copilot instructions, if used
src/payments/AGENTS.md             # optional local rules for a bounded context
src/ml/AGENTS.md                   # optional local rules for AI/RAG subsystem
CLAUDE.md / GEMINI.md              # optional compatibility layer, if team uses these tools
```

**Принцип.** Root `AGENTS.md` — canonical source. Tool-specific files могут ссылаться на него и добавлять форматные требования, но не должны противоречить архитектурным правилам.

**Проверка.**

- Есть ли явный order of precedence?
- Нет ли разных правил для одного и того же поведения?
- Локальные instructions применяются только к своей зоне?
- Tool-specific files не содержат уникальные архитектурные правила, которых нет в canonical docs?

**Red flags.**

- Root `AGENTS.md` говорит «thin handlers», а `.github/copilot-instructions.md` просит писать business logic in route handlers.
- В `CLAUDE.md` есть новые forbidden actions, которых нет в `AGENTS.md`.
- Path-specific file не указывает scope и воспринимается как глобальный.

## D3. Actionable rules

GitHub analysis of agents.md examples подчёркивает, что vague instructions вроде «be helpful» бесполезны; эффективные файлы задают конкретную роль, команды, границы и примеры [E3].

**Actionable instruction содержит:**

```markdown
Bad:
- Write clean code.

Good:
- Keep HTTP handlers thin: parse request, call a use case, map response.
- Do not import `app.infra.*` from `app.domain.*`.
- After changing code under `app/orders`, run:
  - `just test-orders`
  - `just lint`
  - `just typecheck`
```

**Проверка.** Для каждого правила спросить: «Может ли агент выполнить или проверить это правило?» Если нет, правило слишком абстрактное.

## D4. Progressive disclosure

**Что проверяем.** Контекст раскрывается ступенями:

```text
AGENTS.md
  -> README.md
  -> docs/architecture.md
  -> docs/flows/*.md
  -> docs/adr/*.md
  -> local README.md / local AGENTS.md
  -> code + tests
```

**Зачем.** Контекстное окно заполняется файлами, command output и диалогом; по мере заполнения качество работы может снижаться [E5]. Поэтому агент должен читать только релевантные документы, а не грузить весь corpus.

**Red flags.**

- `AGENTS.md` пытается объяснить весь проект.
- README повторяет архитектурные детали, которые уже есть в docs.
- Локальные директории без README, хотя внутри 10+ файлов и несколько responsibility.
- Агенту приходится читать 20 файлов, чтобы понять один bounded context.

## D5. Documentation as interface

Документация должна быть operational interface. Она не просто «рассказывает про проект», а позволяет выполнить задачу.

**Проверяемые типы docs.**

| Документ | Что должен давать агенту |
|----------|--------------------------|
| `README.md` | Назначение проекта, quick start, verify summary, links. |
| `AGENTS.md` | Инструкции и маршрут для AI-агентов. |
| `docs/architecture.md` | Подсистемы, boundaries, entrypoints, storage, integrations. |
| `docs/flows/*.md` | Runtime-пути с файлами, invariants, tests. |
| `docs/adr/*.md` | Почему принято решение, какие alternatives rejected. |
| `DEBUG.md` / `runbooks/` | Проверенные диагностические процедуры, expected outputs. |
| `docs/architecture-rules.md` | Правила, которые должны совпадать с import-linter/tests/CI. |

**Evidence.** Агент должен уметь взять конкретную задачу и пройти путь: relevant docs → relevant files → verify command.

## D6. Single source of truth

**Что проверяем.** Для ключевых фактов есть один основной источник:

- package layout;
- architecture boundaries;
- setup/verify commands;
- migration rules;
- environment variables;
- API contracts;
- prompt/config versions, если проект содержит AI logic;
- RAG corpus policy, если есть RAG.

**Red flags.**

- Три файла дают разные команды запуска тестов.
- Правило миграций описано в README, Confluence export и `AGENTS.md`, но версии расходятся.
- `.env.example` не совпадает с docs/configuration.md.
- ADR говорит «domain не зависит от infra», но import-linter не настроен и код нарушает правило.

**Как чинить.** Выбрать canonical file, остальные заменить ссылками. Для правил, которые можно проверить, добавить executable check.

## D7. ADR, specs and execution flows

**ADR.** Хранят архитектурные решения и trade-offs. Минимальный формат:

```markdown
# ADR-0007: Separate domain use cases from HTTP handlers

Status: accepted
Date: 2026-04-28
Context:
Decision:
Consequences:
Alternatives considered:
Rules to enforce:
- HTTP handlers must not import DB repositories directly.
Verify:
- `lint-imports`
- `pytest tests/architecture/test_handlers.py`
```

**Specs.** Хранят требования и ожидаемое поведение, особенно для сложных фич.

**Execution flows.** Хранят runtime path и invariants. Они полезнее абстрактных диаграмм, когда агент должен менять код.

Google Conductor/context-driven development описывает полезный сдвиг: specs/plans живут рядом с кодом как persistent Markdown artifacts, а не исчезают в chat history [E8].

## D8. Doc-gardening

**Что проверяем.** Документы не должны становиться stale context. Нужны регулярные или автоматические checks.

**Минимальный набор checks.**

- Broken links.
- Ссылки на несуществующие файлы.
- `TODO`, `TBD`, `FIXME`, `WIP` in docs.
- Docs older than N days for actively changed modules.
- Duplicate commands in multiple files.
- `.env.example` vs config docs mismatch.
- Architecture rules doc vs actual linter config mismatch.

**Пример команды.**

```bash
just docs-check
# or
python tools/check_docs.py
```

**Red flags.**

- Документация «красиво написана», но никто не знает, актуальна ли она.
- Модуль менялся 20 раз, а local README не менялся год.
- Runbook не содержит expected outputs и verification step.

## D9. Human README vs agent docs

README обычно нужен людям: purpose, quick start, contribution. `AGENTS.md` нужен агентам: commands, constraints, navigation. Они должны ссылаться друг на друга, но не дублировать всё.

**Правило.**

- README: «что это и как начать».
- AGENTS.md: «как безопасно работать с этим репозиторием».
- Architecture docs: «как устроена система».
- Local docs: «как устроена конкретная зона».

## D10. Conflict resolution

В root `AGENTS.md` должен быть короткий раздел:

```markdown
## Instruction precedence

1. User task and explicit maintainer instructions.
2. This `AGENTS.md`.
3. Local `AGENTS.md` in the directory being edited.
4. Tool-specific instruction files, only when they do not conflict with this file.
5. Older docs only if they are linked from current docs.

If instructions conflict, stop and report the conflict with file paths. Do not silently choose one.
```

Это особенно важно, если проект поддерживает несколько agent tools: GitHub Copilot, Claude Code, Gemini CLI, Codex, Cursor/Windsurf/другие.

## Рекомендуемая структура root `AGENTS.md`

См. `../templates/AGENTS.template.md`. Минимальная структура:

1. Scope and purpose.
2. Project map.
3. Read first.
4. Setup commands.
5. Verify commands.
6. Architecture invariants.
7. Coding rules.
8. Testing rules.
9. Dangerous operations / protected files.
10. Documentation rules.
11. Output expectations.
12. Conflict handling.

## Итоговые вопросы для агента

1. Может ли агент начать работу без устных пояснений?
2. Есть ли короткий root `AGENTS.md`, который ведёт к нужным docs?
3. Нет ли конфликтов между agent instruction files?
4. Документы помогают выполнять работу или только описывают проект?
5. Есть ли single source of truth для архитектуры, verify и environment?
6. Есть ли проверка stale docs?
