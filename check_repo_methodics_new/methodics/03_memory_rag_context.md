# 03. Memory, RAG and Context Management

Методики проверки того, какие знания внутри репозитория стоит считать долговременной памятью, когда нужен RAG/retrieval, и какие memory-практики опасны для кодовой базы.

## Карта методик

| ID | Методика | Главный вопрос |
|----|----------|----------------|
| M1 | Memory taxonomy | Какие виды памяти есть и какие из них должны жить в Git? |
| M2 | Repository as system of record | Репозиторий является source of truth или только одной из копий? |
| M3 | What to store | Какие durable knowledge artifacts стоит хранить внутри repo? |
| M4 | What not to store | Какие memory/retrieval artifacts не стоит коммитить? |
| M5 | Retrieval ladder | Соответствует ли retrieval сложность размеру и боли проекта? |
| M6 | Agent-directed retrieval | Контекст выбирается по задаче или набивается автоматически? |
| M7 | RAG inside repo | Когда RAG полезен, а когда маскирует плохую структуру? |
| M8 | RAG evals | Измеряются ли retrieval relevance и answer faithfulness? |
| M9 | Code graph vs vector RAG | Когда нужен symbol/call/dependency graph? |
| M10 | Context anti-patterns | Какие антипаттерны портят работу агентов? |

## M1. Memory taxonomy

Для репозитория важно разделять разные виды памяти:

| Тип памяти | Где живёт | Что хранить | Риск |
|------------|-----------|-------------|------|
| Durable project memory | Git | ADR, specs, runbooks, architecture docs, eval fixtures | Stale docs, duplication |
| Local scoped instructions | Git | `AGENTS.md`, local `AGENTS.md`, path rules | Конфликты, раздувание |
| Generated retrieval index | Outside Git или reproducible cache | Vector index, symbol graph, embeddings | Шум, секреты, огромные бинарники |
| Session/task memory | Agent session/scratchpad | Текущий план, временные findings | Нельзя считать source of truth |
| Auto memory | Tool-specific store | Повторяющиеся corrections/preferences | Может конфликтовать с repo rules |
| Operational memory | Git + issue tracker | post-mortems, incidents, runbooks | Потеря связи с tests/changes |

Claude Code docs явно разделяют written instructions (`CLAUDE.md`) и auto memory; auto memory — заметки, которые агент пишет сам, а инструкции должны быть конкретными и лаконичными [E6]. Gemini CLI также использует hierarchical/JIT context files и команды `/memory` для управления loaded context [E7]. Эти идеи полезны независимо от конкретного инструмента: **репозиторий должен отличать проверенные правила от авто-сохранённых заметок**.

## M2. Repository as system of record

**Что проверяем.** Важные знания о проекте живут в Git или имеют ссылку из Git на authoritative system.

**Должно быть в Git.**

- Архитектура и boundaries.
- ADR/decision log.
- Specs и acceptance criteria для важных функций.
- Execution flows.
- Runbooks и `DEBUG.md`.
- `.env.example` и configuration contract.
- Evals/golden datasets/fixtures для AI/RAG logic.
- Architecture rules and check configs.
- Scripts/task runner commands.

**Может жить вне Git, но должно быть связано.**

- Production dashboards.
- Incident tracker.
- Secrets manager.
- Long-lived enterprise docs, если они официально canonical.

Если canonical knowledge вне Git, в репозитории нужна ссылка и правило обновления. Иначе агент будет использовать устаревшие локальные копии.

## M3. What to store inside repo as memory

Хранить стоит устойчивые, ревьюируемые, версионируемые знания.

| Artifact | Почему полезен агенту | Условия качества |
|----------|------------------------|------------------|
| `AGENTS.md` | Стартовая навигация и правила | Короткий, actionable, без конфликтов |
| `docs/architecture.md` | System map and boundaries | Совпадает с package layout/imports |
| `docs/flows/*.md` | Runtime context | Entry points, steps, invariants, tests |
| `docs/adr/*.md` | Причины решений | Status, consequences, enforceable rules |
| `runbooks/*.md` / `DEBUG.md` | Диагностика | Проверенные команды, expected outputs |
| `evals/fixtures/*` | Regression memory | Версионируется, имеет owner и thresholds |
| `tools/*.py` / `scripts/*` | Repeatable actions | Исполняется в verify или локально |
| `.importlinter` / architecture checks | Enforced boundaries | Связано с architecture docs |
| `CHANGELOG.md` / release notes | Historical context | Не заменяет ADR, но помогает chronology |
| post-mortems | Operational learning | Есть follow-up changes/tests |

## M4. What not to store inside repo as memory

**Обычно не коммитить:**

- Raw chat logs and transcripts.
- Session scratchpads агента.
- Auto-generated summaries без review.
- Vector index binaries.
- Raw embeddings.
- Large generated dependency graphs, если они легко пересобираются.
- Unredacted logs/traces.
- Secrets, tokens, credentials, `.env`.
- Vendor-specific prompt experiments, если они не являются частью продукта.
- «Память о предпочтениях агента», которая не прошла human review.

**Почему.** Такие артефакты быстро устаревают, шумят retrieval, могут содержать секреты, искажать source of truth и увеличивать стоимость работы агента.

**Разрешённое исключение.** Generated artifact можно хранить, если он мал, детерминирован, нужен offline, имеет команду пересборки и не содержит секретов. Например: compact architecture snapshot, generated API schema, lockfile, small test fixture.

## M5. Retrieval ladder

Для 50–100 KLOC не стоит сразу прыгать в сложный RAG. Сначала проверить, работает ли простая лестница.

| Ступень | Что это | Когда достаточно | Evidence |
|---------|---------|------------------|----------|
| 1 | File tree + `rg` | Малые задачи, ясный layout | Агент быстро находит файлы |
| 2 | Local README/docs | Крупные директории/contexts | Есть локальная навигация |
| 3 | Symbol search/LSP | Много функций/classes | `go to definition`, references работают |
| 4 | Import/dependency graph | Есть cross-boundary complexity | Можно увидеть cycles, fan-in/fan-out |
| 5 | Hybrid search over docs/code | Терминология плавает, много natural-language queries | Есть eval queries |
| 6 | Code graph / RepoGraph-style | Impact analysis дорог, cross-file dependencies важны | Есть graph config + metrics |
| 7 | Full RAG system | Проект сам выдаёт AI-ответы или нужна регулярная NL QA по repo | Есть evals, corpus policy, citations |

**Правило.** Следующую ступень внедряют только если предыдущая не решает конкретную боль. Иначе retrieval становится дорогой маской для плохой структуры.

## M6. Agent-directed retrieval

**Что проверяем.** Агент сам выбирает, какие документы открыть, по задаче и evidence. Контекст не набивается автоматически всем, что кажется полезным.

**Хороший workflow.**

1. Read `AGENTS.md`.
2. Identify relevant bounded context.
3. Read local README/AGENTS.
4. Use grep/symbol search for concrete entities.
5. Open only files on suspected change path.
6. Run targeted verify.
7. Expand context only when evidence требует.

**Плохой workflow.**

1. Load all docs.
2. Load all files from domain.
3. Add generated summaries.
4. Add chat history.
5. Ask model to infer.

Context stuffing повышает шум и провоцирует противоречивые выводы, особенно если docs stale.

## M7. RAG inside repo: when useful / when harmful

### RAG полезен, если

- В проекте есть AI/RAG feature и нужно проверять его как часть продукта.
- В docs много versioned knowledge, которое не помещается в обычный context.
- Команда часто задаёт natural-language вопросы по проекту.
- Термины расходятся: user language ≠ code symbols.
- Нужен traceable answer with citations to files/docs.
- Есть eval queries и пороги качества.

### RAG вреден или преждевременен, если

- Нет `AGENTS.md`, architecture map и verify loop.
- Docs stale или противоречат коду.
- В Git нет stable source-of-truth artifacts.
- Команда пытается индексом заменить нормальные boundaries.
- Retrieval возвращает большие chunks без path/line/symbol attribution.
- Нет измерения retrieval relevance и answer faithfulness.
- Индекс хранит secrets, logs, chat transcripts или generated noise.

## M8. RAG evals

Если проект содержит RAG или внутренний repo-RAG, качество нужно разделять на retrieval и generation.

**Retrieval quality.**

- Context relevance.
- Recall@k for known queries.
- MRR / nDCG для ranked results.
- Chunk attribution: path, line range, symbol.
- Coverage of important docs and code zones.

**Generation quality.**

- Faithfulness: claims поддержаны retrieved context [E11].
- Answer relevance: ответ отвечает на вопрос.
- Citation coverage: важные утверждения привязаны к источнику.
- Format validity: JSON/schema/Markdown contract.
- Safety: не раскрывает секреты, не предлагает опасные действия.

Ragas docs описывают objective metrics для LLM applications/RAG и agentic workflows [E11][E12]. LangChain/LangSmith docs также подчёркивают необходимость eval-контура для RAG-приложений [E13].

**Минимальный набор для repo-RAG.**

```text
evals/rag_queries.yml
- question: "Where is payment callback idempotency enforced?"
  expected_sources:
    - docs/flows/payment_callback.md
    - app/domain/orders/payment.py
  must_not_sources:
    - old_docs/payments_legacy.md
  checks:
    - retrieval_recall_at_5
    - answer_faithfulness
    - citation_required
```

## M9. Code graph vs vector RAG

Vector/hybrid search полезен для semantic recall, но плохо гарантирует structural understanding. Для кода важны symbol references, imports, calls, class hierarchy, dependency direction, ownership and file locality.

**Когда нужен code graph.**

- Агент часто правит cross-file behavior.
- Ошибки impact analysis дороги.
- Есть много adapters/providers with similar names.
- Simple grep находит слишком много ложных совпадений.
- Нужно понимать multi-hop dependencies.

RepoGraph-style подходы предлагают repository-level structure как навигацию для AI software engineering [E23]. Но graph — не обязательная первая мера. Для 50–100 KLOC сначала часто достаточно import graph + LSP/symbol search + локальных docs.

## M10. Context anti-patterns

| Антипаттерн | Признак | Что делать |
|-------------|---------|------------|
| Context stuffing | В prompt/index грузится весь corpus | Progressive disclosure, agent-directed retrieval |
| Stale context | Docs старые, но попадают в retrieval | Doc-gardening, stale markers, canonical docs |
| Duplicate context | Одно правило в 5 местах | Single source of truth + links |
| Context leakage | В instructions/logs попали secrets | Secret scanning, redaction, no raw logs |
| No context | Нет карты проекта/flows | Создать AGENTS + architecture map |
| Over-indexing | Большие vector dumps в Git | Keep generated indexes outside Git |
| Summary drift | Generated summaries не совпадают с кодом | Human review или не хранить summaries |
| Tool memory override | Auto memory конфликтует с repo rules | Repo rules имеют priority, конфликт репортится |
| Long-context complacency | «Модель видит всё» вместо нормального layout | Нарезка модулей, локальные docs, boundaries |

## Итоговые вопросы для агента

1. Какие знания в проекте являются durable memory, а какие временным scratchpad?
2. Есть ли single source of truth для архитектуры, flows, setup, verify, env?
3. Не хранит ли repo raw chat logs, embeddings, vector dumps, unredacted logs?
4. Соответствует ли retrieval ladder реальному размеру и боли проекта?
5. Есть ли RAG evals, если RAG используется?
6. Отделён ли semantic search от structural understanding?
7. Может ли агент объяснить, почему он открыл именно эти файлы?
