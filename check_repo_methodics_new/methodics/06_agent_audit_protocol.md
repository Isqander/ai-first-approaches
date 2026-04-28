# 06. Agent Audit Protocol

Пошаговый протокол для AI-агента, который проверяет репозиторий по этой методике. Протокол помогает не уходить в реализацию раньше аудита и не делать выводы без evidence.

## Режим работы

Агент должен действовать как auditor/improvement planner, а не как фича-разработчик. До завершения аудита не менять код, кроме случаев, когда задача явно просит сразу исправлять и это безопасно. Если пользователь просит улучшить репозиторий, сначала собрать findings и предложить порядок изменений.

## Фаза 0. Ограничения и scope

Зафиксировать:

- язык и размер проекта: Python, примерно 50–100 KLOC;
- что входит в аудит: repo docs, instructions, architecture, code layout, verify, tests, memory/RAG, evals, observability, governance;
- что не входит: выбор модели, agent harness, IDE, workflow prompts, MCP selection;
- какие parts не применимы: например, AI/RAG observability не нужна, если проект не AI-система.

## Фаза 1. Быстрая инвентаризация

Собрать карту файлов без чтения всего проекта.

Пример команд, которые агент может адаптировать:

```bash
pwd
find . -maxdepth 3 -type f | sed 's#^./##' | sort | head -300
find . -maxdepth 2 -type f \( -name 'README*' -o -name 'AGENTS.md' -o -name 'CLAUDE.md' -o -name 'GEMINI.md' -o -name 'pyproject.toml' -o -name 'justfile' -o -name 'Makefile' -o -name 'tox.ini' -o -name 'noxfile.py' \) -print
```

Собрать список docs:

```bash
find docs -maxdepth 3 -type f 2>/dev/null | sort
find . -maxdepth 4 -type f \( -iname '*architecture*' -o -iname '*adr*' -o -iname '*flow*' -o -iname '*debug*' -o -iname '*runbook*' \) | sort
```

Собрать Python hotspots:

```bash
python - <<'PY'
from pathlib import Path
files=[]
for p in Path('.').rglob('*.py'):
    if any(part in {'.git','.venv','venv','site-packages','__pycache__'} for part in p.parts):
        continue
    try:
        n=len(p.read_text(errors='ignore').splitlines())
    except Exception:
        continue
    files.append((n,str(p)))
for n,p in sorted(files, reverse=True)[:30]:
    print(f'{n:5} {p}')
print('python_files=', len(files), 'loc=', sum(n for n,_ in files))
PY
```

## Фаза 2. Прочитать стартовый контекст

Читать в таком порядке:

1. `AGENTS.md`, если есть.
2. Root `README.md`.
3. `pyproject.toml` and task runner files.
4. `docs/architecture.md` or closest equivalent.
5. `docs/flows/*` for critical flows.
6. `docs/adr/*` index/recent decisions.
7. Local README/AGENTS for the zones touched by findings.

Не читать весь проект подряд. Расширять контекст только по evidence.

## Фаза 3. Проверить agent instructions

Ответить:

- Есть ли root `AGENTS.md`?
- Это карта или энциклопедия?
- Есть ли setup, test, verify commands?
- Есть ли architecture invariants and forbidden actions?
- Есть ли local/path-specific instructions?
- Не конфликтуют ли tool-specific files?
- Есть ли explicit precedence?
- Есть ли output expectations for agents?

Evidence фиксировать путями и короткими цитатами/пересказами.

## Фаза 4. Проверить architecture/code alignment

Построить таблицу:

| Claimed subsystem | Docs path | Code path | Tests | Import rule | Notes |
|-------------------|-----------|-----------|-------|-------------|-------|

Проверить:

- docs map vs directory tree;
- entrypoints;
- domain/infra/api/workers boundaries;
- public API;
- cross-boundary imports;
- hotspots and god files;
- side effects and import-time behavior;
- generated code rules.

Если architecture docs нет, это finding `A1/A2=0`, но агент всё равно должен reconstruct provisional map from file tree and imports.

## Фаза 5. Проверить verify loop

Найти:

- full verify command;
- partial commands;
- CI equivalent;
- architecture/import checks;
- docs checks;
- security/dependency checks;
- AI/RAG eval commands, if relevant.

Если команда есть, не обязательно запускать всё в тяжёлом проекте. Но нужно проверить, что команда существует, documented and plausible. Если запускается — фиксировать результат.

Пример evidence:

```text
E1 full verify: `just verify` exists in justfile and runs `ruff check`, `mypy`, `pytest`, `lint-imports`.
E2 no docs check: no command found for broken links/stale docs.
E3 CI mismatch: GitHub Actions runs `pytest --cov`, local docs only mention `pytest`.
```

## Фаза 6. Проверить memory/RAG/context

Ответить:

- Где durable knowledge? `docs/adr`, `docs/flows`, runbooks, post-mortems?
- Есть ли duplicate/stale docs?
- Есть ли raw chat logs/scratchpads/vector dumps in repo?
- Есть ли retrieval/index config?
- Если RAG есть: что индексируется, как исключаются secrets, где evals?
- Есть ли separate metrics for retrieval and generation?

Для проекта без AI/RAG логики advanced RAG может быть `N/A`, но memory hygiene всё равно оценивается.

## Фаза 7. Проверить observability/governance/security

Для любого проекта:

- `.env.example` and secrets policy;
- branch protection / required checks if visible;
- dependency audit command;
- CODEOWNERS for critical files;
- security docs or runbooks;
- architecture gates.

Для AI/RAG проекта дополнительно:

- traces/logs for model/tool/retrieval runs;
- prompt/config lineage;
- eval datasets;
- error taxonomy;
- HITL/approval thresholds;
- incident feedback loop;
- mapping or readiness for OTel GenAI conventions.

## Фаза 8. Заполнить чек-лист

Использовать `../CHECK_REPO_AUDIT_CHECKLIST.md`.

Правила:

- Не ставить `2` без evidence.
- `1` означает «частично есть, но есть существенный риск/дырка».
- `N/A` допустимо только для AI/RAG-specific пунктов, если проект действительно не содержит AI-output/RAG/agent runtime.
- Если `N/A`, пересчитать score для применимой части отдельно.
- Каждый `0` и `1` должен попадать в backlog или explicit non-goal.

## Фаза 9. Сформировать backlog

Backlog должен быть actionable. Формат:

| Priority | Item | Evidence | Proposed change | Expected impact | Effort | Risk |
|----------|------|----------|-----------------|-----------------|--------|------|
| P0 | Add root `AGENTS.md` | no file found | create file from template | faster onboarding, fewer wrong commands | S | Low |
| P1 | Enforce domain/infra boundary | domain imports infra db | add import-linter contract, refactor 3 imports | reduces architecture drift | M | Medium |

Порядок исправлений:

1. Instructions and docs map.
2. Verify command and fast loop.
3. Architecture boundaries and import checks.
4. Hotspot refactoring.
5. Doc-gardening.
6. AI/RAG evals/observability, if applicable.
7. Advanced retrieval/code graph only after base is stable.

## Фаза 10. Формат итогового отчёта

Использовать `../templates/CHECK_REPO_REPORT.template.md`.

Минимальный отчёт:

```markdown
# CHECK_REPO_REPORT

## Summary
- Total score:
- Readiness level:
- Top 3 strengths:
- Top 3 risks:

## Section scores
| Section | Score | Max | Notes |

## P0 findings
...

## Evidence-based findings
| ID | Score | Evidence | Recommendation |

## Backlog
...

## N/A and non-goals
...
```

## Decision rules для рекомендаций

### Не рекомендовать RAG, если

- нет `AGENTS.md`;
- нет architecture map;
- нет verify command;
- docs stale/duplicated;
- агент не может найти код через tree/grep/docs.

В этом случае сначала чинить structure/docs/verify.

### Не рекомендовать большой рефакторинг, если

- нет architecture rules;
- нет tests around changed area;
- нет rollback path;
- неясно, какие flows затронуты.

Сначала добавить tests/rules/docs around target.

### Не рекомендовать новые agent instructions, если

- они дублируют existing rules;
- они tool-specific без canonical source;
- они не actionable;
- они конфликтуют с docs/code.

Сначала сделать canonical root instruction and references.

### Не считать отсутствие AI observability проблемой, если

- проект не использует AI-output/RAG/agent runtime;
- LLM применяется только внешним разработчиком, а не как часть продукта.

Но verify/docs/architecture всё равно должны быть хорошими для agents.

## Типовые findings и быстрые улучшения

| Finding | Fast improvement |
|---------|------------------|
| Нет `AGENTS.md` | Создать root file from template, 80–150 lines. |
| Нет verify command | Добавить `just verify`/`make verify`, объединить existing checks. |
| Docs contradict commands | Выбрать canonical command, заменить остальные ссылками. |
| Domain imports infra | Добавить forbidden import rule, refactor via port/interface. |
| God file | Сначала вынести local README/tests, затем extract cohesive module. |
| Stale runbook | Добавить expected outputs and last verified date. |
| RAG без evals | Добавить 20–50 representative queries before tuning retrieval. |
| Prompt hidden in code | Move to versioned config/template and log version. |
| Secrets risk | `.env.example`, secret scanning/push protection, docs/configuration.md. |

## Quality bar for the auditor agent

Хороший аудит:

- не абстрактный;
- не предлагает «использовать лучший AI»;
- показывает file paths;
- разделяет P0/P1/P2/P3;
- отделяет base hygiene от advanced AI-first features;
- не внедряет RAG вместо нормальной структуры;
- учитывает, что Python 50–100 KLOC требует explicit package/import/test/tooling discipline;
- даёт план, который можно превратить в PR series.
