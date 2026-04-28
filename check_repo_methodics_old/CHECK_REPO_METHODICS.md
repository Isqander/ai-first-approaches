# CHECK_REPO_METHODICS

> Навигационный файл для аудита AI-friendly Python-репозитория размером 50–100 KLOC.

## Область применимости

Методика рассчитана прежде всего на репозиторий, с которым работает именно AI-агент, а не только чат-интерфейс. Если LLM используется исключительно как чат без tool use, verify-loop и автономных действий, часть требований про retrieval, observability, governance и feedback loop может быть неприменима — это нужно явно отметить при аудите.

Секции `A–G` в чек-листе в основном language-agnostic. Секция `H` — Python-специфичная адаптация для репозитория 50–100 KLOC.

## Зачем документ разбит на несколько файлов

Для AI-агента полезнее progressive disclosure, чем один длинный файл. Поэтому `CHECK_REPO_METHODICS.md` теперь играет роль карты, а детали вынесены в тематические документы.

Такой формат даёт три преимущества:

- агент быстрее находит нужный слой проверки;
- снижается шум в рабочем контексте;
- проще использовать документ и как методику проектирования, и как реальный чек-лист аудита.

## Как пользоваться

- если нужно понять подход целиком, начните с `check_repo_methodics/01_methodics_architecture_and_code.md`;
- если задача про `AGENTS.md`, docs, память и retrieval, читайте `check_repo_methodics/02_methodics_docs_memory_and_rag.md`;
- если задача про verify, evals, runbooks, observability и governance, читайте `check_repo_methodics/03_methodics_verify_observability_and_governance.md`;
- если нужно провести сам аудит и выставить оценки, используйте `check_repo_methodics/04_audit_checklist.md`.

Быстрый маршрут для AI-агента:

- открыть `check_repo_methodics/04_audit_checklist.md` и отметить слабые зоны;
- для каждого проваленного блока перейти в тематический файл;
- зафиксировать evidence, а не только score;
- после аудита сформировать backlog в порядке: structural alignment -> docs/context -> verify -> advanced retrieval/observability.

## Принцип оценки

Для каждого пункта используется шкала:

- `0` — отсутствует, противоречиво или вредно устроено;
- `1` — есть частично, но не системно или не проверяется;
- `2` — есть, используется и поддерживается автоматически или организационно.

Рекомендуемая интерпретация:

- `85%+` от максимума — репозиторий хорошо приспособлен для AI-агентов;
- `65–84%` — работать можно, но агент будет тратить много времени на восстановление контекста;
- `45–64%` — AI полезен локально, но системная скорость и надёжность ограничены;
- `<45%` — сначала нужно чинить структуру репозитория и документацию, потом наращивать агентность.

## Карта документов

- `check_repo_methodics/01_methodics_architecture_and_code.md` — архитектура, structural alignment, границы модулей, AI-friendly код;
- `check_repo_methodics/02_methodics_docs_memory_and_rag.md` — `AGENTS.md`, documentation as interface, memory, retrieval, RAG;
- `check_repo_methodics/03_methodics_verify_observability_and_governance.md` — verify-контур, fitness functions, runbooks, evals, observability, governance;
- `check_repo_methodics/04_audit_checklist.md` — компактный аудит-лист со score и evidence.

## Минимальный обязательный набор

Если нужно быстро довести Python-репозиторий 50–100 KLOC до рабочего AI-friendly состояния, сначала нужны именно эти вещи:

- короткая карта архитектуры и 3–5 execution flows;
- `AGENTS.md` как оглавление проекта;
- структура пакетов, совпадающая с реальными доменными границами;
- публичные контракты на ключевых функциях и сервисах;
- единая команда `verify`;
- проверенные `runbooks` или `DEBUG.md` для критических сценариев;
- единый source of truth для docs, ADR и проектных правил;
- хотя бы минимальные архитектурные тесты или fitness functions.

## Порядок исправлений

Если аудит выявил много проблем сразу, полезно исправлять их в таком порядке:

1. AI-instructions: `AGENTS.md` и связанные instruction files — самый быстрый выигрыш для нового агента.
2. Structural alignment и нарезка god files — уменьшают шум и локализуют изменения.
3. Контракты на публичных API и явные зависимости — убирают скрытый контекст.
4. Тесты рядом с кодом и единая команда `verify` — ускоряют feedback loop.
5. Doc-gardening, retrieval, observability и advanced governance — усиливают зрелость после стабилизации базовой структуры.

## Что документ не покрывает

Этот набор методик не предназначен для выбора:

- моделей и провайдеров;
- агентного харнесса;
- конкретных рабочих промптов;
- orchestration вне репозитория.

Он оценивает именно сам проект как среду, в которой AI-агент читает, понимает, меняет и проверяет код.

## Источники внутри репозитория

- `ArchitectureAsCode/partitioned/01_chapter1_part1.md`
- `ArchitectureAsCode/partitioned/02_chapter1_part2.md`
- `ArchitectureAsCode/partitioned/06_chapter3_part2.md`
- `ArchitectureAsCode/partitioned/07_chapter4_part1.md`
- `ArchitectureAsCode/partitioned/10_chapter5_part1.md`
- `ai-first-approaches/02-context-engineering.md`
- `ai-first-approaches/03-agent-driven-development.md`
- `ai-first-approaches/04-observability-and-evals.md`
- `ai-first-approaches/05-knowledge-and-memory.md`
- `ai-first-approaches/06-codebase-as-prompt.md`
- `ai-first-approaches/08-security-and-governance.md`
- `book/10_агент_не_чат.md`
- `book/12_RAG.md`
- `book/14_оценка_качества_llm_систем.md`
- `book/16_архитектура_кода.md`
- `book/17_наблюдаемость_и_эксплуатация.md`
