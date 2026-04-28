# 04. Verify, Evals, Observability and Governance

Методики проверки того, умеет ли проект подтверждать изменения, ловить architectural drift, измерять AI/RAG-качество и удерживать безопасные границы.

## Карта методик

| ID | Методика | Главный вопрос |
|----|----------|----------------|
| V1 | Single verify entrypoint | Можно ли одной командой прогнать обязательные проверки? |
| V2 | Fast local feedback | Есть ли быстрые partial checks для итераций агента? |
| V3 | Scripts over prose | Повторяемые действия формализованы как команды? |
| V4 | Architecture fitness functions | Архитектурные свойства проверяются автоматически? |
| V5 | Test topology | Тесты помогают локализовать изменения? |
| V6 | Actionable failures | Ошибки объясняют, что чинить и где? |
| V7 | AI eval contract | Если есть AI-логика, определено ли «хорошо» и как измерять? |
| V8 | Versioned datasets | Golden datasets/fixtures версионируются и связаны с кодом? |
| V9 | Structured observability | Есть traces/logs/metrics, если проект сам AI-система? |
| V10 | OTel GenAI readiness | Есть ли mapping на `gen_ai.*` conventions или план? |
| V11 | Prompt/config lineage | Prompt/config versions связаны с traces и evals? |
| V12 | Error taxonomy | Ошибки AI/RAG разделяются по слоям? |
| V13 | Human review thresholds | HITL включается по риску/uncertainty, а не тотально? |
| V14 | Incident feedback loop | Инциденты превращаются в tests, rules, docs, datasets? |
| V15 | Governance gates | Критичные правила enforced инфраструктурой? |
| V16 | Security baseline | Secrets, dependencies, supply chain, branch protection покрыты? |

## V1. Single verify entrypoint

**Что проверяем.** Есть единая команда полной проверки:

```bash
just verify
make verify
tox
nox -s verify
python -m tools.verify
```

**Минимальный состав.**

- Format/lint check.
- Typecheck.
- Unit tests.
- Integration/smoke tests where practical.
- Architecture/import checks.
- Docs checks for broken links/stale critical docs.
- Security baseline: dependency audit/secret scan if configured.

**Зачем агенту.** Агент не должен реконструировать ritual из README, CI YAML и устных правил. Единая команда делает feedback repeatable.

**Red flags.**

- CI есть, но локальной команды нет.
- README говорит `pytest`, CI запускает ещё 6 скрытых проверок.
- Full verify занимает час, а быстрых partial checks нет.

## V2. Fast local feedback

Агент должен быстро итерироваться. Нужны отдельные команды:

```bash
just lint
just typecheck
just test-unit
just test-path app/orders
just test-smoke
just architecture-check
just docs-check
```

**Проверка.** Для типового изменения в одном bounded context можно запустить targeted verify за разумное время. Важно не абсолютное число секунд, а наличие predictable path: targeted → full.

## V3. Scripts over prose

Если действие повторяется, оно должно быть командой, а не абзацем prose.

**Плохой формат.**

```markdown
Run ruff, then mypy, then pytest, but skip integration tests unless Redis is running.
```

**Хороший формат.**

```bash
just verify
just verify-no-integration
just test-integration
```

**Проверка.** В docs каждая процедура должна иметь команду, expected output или link на script.

## V4. Architecture fitness functions

Fitness functions — объективные проверки архитектурных свойств. Они могут быть простыми: 10 строк Python, import-linter config, pytest, shell script, CI rule. Важно, что architecture intent становится executable feedback, а не пожеланием в Markdown [E25][E26].

**Типы checks.**

| Свойство | Возможный check |
|----------|-----------------|
| Layering | import-linter layers contract |
| Forbidden imports | import-linter forbidden contract / custom pytest |
| No cycles | import graph script |
| Public API boundaries | forbid imports from `_internal`, `impl`, adapters |
| File size | script threshold |
| Complexity | Ruff/flake8 plugin/radon/Sonar/custom script |
| Config hygiene | `.env.example` vs settings schema |
| Docs drift | links and stale markers |
| Prompt/config lineage | prompt files have version IDs, traces include IDs |

**Правило.** Каждая architecture rule в docs должна иметь одно из состояний:

- `enforced`: проверяется автоматически;
- `reviewed`: проверяется на code review по explicit checklist;
- `aspirational`: цель, но не gate.

Если статус не указан, агент не знает, насколько правило обязательно.

## V5. Test topology

**Что проверяем.** Тесты помогают агенту выбирать минимальный verify.

Хорошая топология:

```text
app/orders/...
tests/unit/orders/...
tests/integration/orders/...
docs/flows/order_checkout.md -> links to tests
```

Или тесты colocated, если команда так решила:

```text
app/orders/payment.py
app/orders/test_payment.py
```

Главное — предсказуемость и link между code zone, flow docs and tests.

**Red flags.**

- Все tests в `tests/test_all.py`.
- Названия тестов не отражают домен.
- Integration tests не помечены markers.
- Агент не может понять, какие tests запускать после локального change.

## V6. Actionable failures

**Что проверяем.** Ошибки lint/CI/runtime говорят не только «что сломалось», но и «как чинить».

**Плохая ошибка.**

```text
Forbidden dependency
```

**Хорошая ошибка.**

```text
Architecture rule failed: app.domain must not import app.infra.db.
Move DB access behind app.domain.ports.OrderRepository and implement it in app.infra.db.orders.
See docs/architecture-rules.md#domain-infra-boundary.
```

Для агентов это критично: failure message является следующим prompt-like feedback.

## V7. AI eval contract

Если проект содержит AI/RAG/agent runtime, обычные tests не покрывают quality. Нужен eval contract.

**Минимальное содержание.**

- Core use cases.
- Golden dataset/examples.
- Metrics.
- Thresholds.
- Evaluation command.
- Versioning: model/prompt/config/retrieval corpus.
- How to update dataset.
- Human review criteria.

**Пример.**

```yaml
scenario: answer_repo_question
command: just eval-rag
metrics:
  retrieval_recall_at_5: ">=0.80"
  faithfulness: ">=0.90"
  answer_relevance: ">=0.85"
  citation_coverage: "100% for factual claims"
regression_policy:
  fail_ci_on_core_regression: true
  allow_known_failures_file: evals/known_failures.yml
```

Ragas and related eval frameworks provide metrics for RAG/agent workflows, включая faithfulness и context relevance [E11][E12].

## V8. Versioned datasets and fixtures

**Что проверяем.** Eval datasets, golden examples, regression fixtures лежат рядом с кодом или имеют versioned pointer.

**Хороший минимум.**

```text
evals/
  README.md
  rag_queries.yml
  agent_tasks.yml
  expected_outputs/
  known_failures.yml
  schema.json
```

**Правила.**

- Dataset changes reviewed like code.
- Есть owner.
- Есть reason для добавления/удаления кейса.
- Известно, какие product risks покрываются.
- Не хранить sensitive user data без redaction и legal basis.

## V9. Structured observability

Если проект сам AI-система или содержит agent/RAG runtime, нужны structured traces/logs/metrics.

**Минимальные поля для AI/RAG runs.**

- `trace_id`, `run_id`, `session_id`.
- `scenario` / `use_case`.
- `prompt_version` / `config_sha`.
- `model_provider`, `model_name` if allowed by policy.
- `retriever_version`, `corpus_version`.
- `tool_name`, `tool_result_status`.
- `latency_ms`, `tokens_in`, `tokens_out`, `cost_estimate`.
- `error_category`.
- `quality_score` or eval result when available.

**Red flags.**

- Логи — просто текстовые строки без correlation ID.
- Нельзя связать плохой ответ с версией prompt/retriever/corpus.
- Cost/latency/quality не разделяются по scenario.

## V10. OTel GenAI readiness

OpenTelemetry GenAI semantic conventions define signals for generative AI events, exceptions, metrics, model spans and agent spans; на момент доступа статус conventions указан как development/experimental transition [E9].

**Что проверяем.**

- Есть ли mapping локальных spans/log fields на `gen_ai.*` concepts.
- Не захардкожена ли ad hoc schema без плана миграции.
- Есть ли policy по захвату prompt/response content: opt-in, redaction, privacy.
- Для tool calls есть отдельные spans/events.

**Практичный подход.** Если проект не production AI-system, достаточно structured logs. Если production — полезно заранее проектировать schema так, чтобы потом перейти на OTel GenAI conventions.

## V11. Prompt/config lineage

Даже если эта методика не оценивает «какие промпты писать», она оценивает, **версируются ли prompt/config artifacts**, если они часть продукта.

**Что проверяем.**

- Prompt templates не спрятаны в случайных строках кода.
- Есть `prompts/`, `configs/`, `policies/` или понятный source.
- Версия prompt/config попадает в traces/evals.
- Изменение prompt/config проходит review.
- Есть rollback path.

**Red flags.**

- Prompt изменили, качество упало, но trace не показывает какую версию использовали.
- В evals нет `prompt_version`.
- Prompt/config живут вне Git без versioned references.

## V12. Error taxonomy

Для AI/RAG-систем все ошибки нельзя сваливать в «LLM failed». Минимальная taxonomy:

| Category | Примеры |
|----------|---------|
| Provider | timeout, rate limit, unavailable, quota |
| Tool | tool failed, invalid args, permission denied |
| Retrieval | no relevant chunks, stale chunk, low recall, bad rerank |
| Format | invalid JSON/schema, missing required field |
| Content | hallucination, off-topic, unsupported claim |
| Safety | prompt injection, policy violation, secret exposure |
| Cost | token budget overflow, runaway loop |
| Governance | approval required, forbidden operation |

Эта taxonomy должна попадать в logs, dashboards, incident docs and eval reports.

## V13. Human review thresholds

Human-in-the-loop полезен, если включается по риску. Тотальное ревью всего потока не масштабируется, а отсутствие review опасно.

**Criteria.**

- Low confidence / low eval score.
- Safety-sensitive action.
- Writes to production or external systems.
- Security/privacy impact.
- Large diff or architecture boundary changes.
- New dependency/provider.
- RAG answer lacks citations for factual claims.

**Evidence.** Есть policy, thresholds and review queue/approval path.

## V14. Incident feedback loop

**Что проверяем.** Инциденты и серьёзные bugs возвращаются в проект:

```text
incident -> post-mortem -> root cause -> new/updated test -> updated docs/runbook -> updated agent instruction if needed -> verify gate
```

**Red flags.**

- Постмортем есть, но ни один test/rule/doc не изменился.
- Одинаковый инцидент повторяется под новым названием.
- Runbook не обновлён после изменения production behavior.

## V15. Governance gates

**Что проверяем.** Критичные правила enforced инфраструктурой.

| Rule | Better enforcement |
|------|--------------------|
| Do not commit secrets | Secret scanning, push protection, pre-commit, CI |
| Domain must not import infra | import-linter / architecture tests |
| Generated code must be reviewed | branch protection / PR labels / CODEOWNERS |
| Dangerous migrations require review | migration linter + CODEOWNERS |
| New dependency requires approval | dependency review / SBOM / SCA |
| AI-output must be grounded | eval threshold + citation requirement |

OWASP LLM Top 10 highlights risks such as prompt injection, insecure output handling, sensitive information disclosure and supply-chain vulnerabilities [E10]. Security rules should therefore be gates where practical, not prose-only.

## V16. Security baseline

Для репозитория 50–100 KLOC минимальный security/governance baseline:

- `.env.example`, no real secrets.
- Secret scanning / push protection where available; GitHub docs describe push protection as blocking credentials before they reach the repo [E22].
- Dependency scanning/SCA or at least dependency audit command.
- Lockfiles or pinned dependency policy.
- Branch protection for required checks.
- CODEOWNERS for critical areas.
- SBOM/provenance approach if project is shipped externally.
- Supply-chain checklist aligned with SSDF/SLSA concepts where relevant [E19][E20][E21].
- LLM-specific safety notes if project uses model outputs/tools/RAG.

## Итоговые вопросы для агента

1. Можно ли одной командой проверить обязательный baseline?
2. Есть ли быстрые partial checks?
3. Какие архитектурные правила enforced, а какие только documented?
4. Если есть AI/RAG, чем измеряется качество?
5. Можно ли связать плохой AI-result с prompt/config/retriever/corpus versions?
6. Есть ли error taxonomy и incident feedback loop?
7. Какие safety rules enforced инфраструктурой?
