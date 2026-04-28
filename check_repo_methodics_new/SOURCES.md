# SOURCES

Дата доступа к внешним источникам: 2026-04-28.

## Внешние источники

| ID | Источник | Для чего использован |
|----|----------|----------------------|
| E1 | AGENTS.md — https://agents.md/ | Базовая идея `AGENTS.md` как README для coding agents: setup, tests, conventions, project-specific guidance. |
| E2 | GitHub Docs, repository custom instructions — https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/add-custom-instructions/add-repository-instructions | Иерархия repository-wide/path-specific/agent instructions, вложенные `AGENTS.md`, precedence nearest file, поддержка `CLAUDE.md`/`GEMINI.md`. |
| E3 | GitHub Blog, “How to write a great agents.md” — https://github.blog/ai-and-ml/github-copilot/how-to-write-a-great-agents-md-lessons-from-over-2500-repositories/ | Практические признаки хороших agent files: конкретная роль, команды, границы, примеры, запреты. |
| E4 | OpenAI, “A practical guide to building agents” — https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/ | Общая инженерная рамка: агент как система с tools, guardrails, orchestration и predictable/safe execution. Не используется для выбора модели или харнесса. |
| E5 | Anthropic Claude Code, Best Practices — https://code.claude.com/docs/en/best-practices | Контекстное окно заполняется файлами/командами/диалогом; agentic coding отличается от чат-режима. |
| E6 | Anthropic Claude Code, Memory — https://code.claude.com/docs/en/memory | Разделение persistent instructions (`CLAUDE.md`) и auto memory; scoped rules; keep instructions concise/specific. |
| E7 | Gemini CLI, `GEMINI.md` context files — https://geminicli.com/docs/cli/gemini-md/ | Hierarchical/JIT context files, `/memory show/reload/add`, контекст как слой, а не одноразовый prompt. |
| E8 | Google Developers Blog, Conductor/context-driven development — https://developers.googleblog.com/conductor-introducing-context-driven-development-for-gemini-cli/ | Спецификации и планы как persistent Markdown-артефакты рядом с кодом; репозиторий как source of truth для агентной работы. |
| E9 | OpenTelemetry GenAI semantic conventions — https://opentelemetry.io/docs/specs/semconv/gen-ai/ | GenAI signals: events, exceptions, metrics, model spans, agent spans; статус development/experimental. |
| E10 | OWASP Top 10 for LLM Applications — https://owasp.org/www-project-top-10-for-large-language-model-applications/ | Риски LLM-приложений: prompt injection, insecure output handling, data poisoning, DoS, supply-chain, sensitive info, insecure plugins. |
| E11 | Ragas Faithfulness metric — https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/faithfulness/ | Метрика faithfulness: все claims ответа должны поддерживаться retrieved context. |
| E12 | Ragas available metrics — https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/ | Наборы метрик для RAG и agentic workflows; objective measurement для LLM apps. |
| E13 | LangChain/LangSmith, Evaluate a RAG application — https://docs.langchain.com/langsmith/evaluate-rag-tutorial | RAG как техника external knowledge + важность eval-контура для RAG-приложений. |
| E14 | Python Packaging User Guide, `pyproject.toml` — https://packaging.python.org/en/latest/guides/writing-pyproject-toml/ | `pyproject.toml` как единая точка для build-system/project/tool tables. |
| E15 | Ruff Formatter docs — https://docs.astral.sh/ruff/formatter/ | `ruff format` как entrypoint и единый toolchain for lint/format. |
| E16 | mypy docs — https://mypy.readthedocs.io/en/stable/ | Gradual typing и статическая проверка типов для Python. |
| E17 | Import Linter docs — https://import-linter.readthedocs.io/en/stable/ | Ограничения import dependencies между Python-модулями. |
| E18 | Import Linter layers contract — https://import-linter.readthedocs.io/en/latest/contract_types/layers/ | Layers contract: higher layers may depend on lower, not reverse; indirect imports count. |
| E19 | NIST SSDF SP 800-218 v1.1 — https://csrc.nist.gov/pubs/sp/800/218/final | Secure software development как набор high-level practices; common vocabulary. |
| E20 | NIST SSDF SP 800-218 Rev.1 initial public draft — https://csrc.nist.gov/pubs/sp/800/218/r1/ipd | 2025 draft with improved practices/tasks/examples; использовать как ориентир, не как final standard. |
| E21 | SLSA / OpenSSF — https://openssf.org/projects/slsa/ и https://slsa.dev/ | Supply-chain framework/checklist: tampering prevention, provenance, integrity, checklist vocabulary. |
| E22 | GitHub Push Protection docs — https://docs.github.com/en/code-security/concepts/secret-security/about-push-protection | Блокировка hardcoded credentials before they reach repository. |
| E23 | RepoGraph, arXiv/ICLR 2025 — https://arxiv.org/abs/2410.14684 | Repository-level code graph as navigation/context for AI software engineering. |
| E24 | Architecture Without Architects, arXiv 2026 — https://arxiv.org/html/2604.04990v1 | AI coding agents can make implicit architecture decisions; architecture needs review/enforcement. |
| E25 | Thoughtworks, Architecture as Code podcast, 2025 — https://www.thoughtworks.com/insights/podcasts/technology-podcasts/architecture-as-code | Fitness functions and architecture-as-code as concrete verification of architecture intent. |
| E26 | InfoQ, Fitness Functions for Your Architecture, 2025 — https://www.infoq.com/articles/fitness-functions-architecture/ | Fitness functions as guardrails/objective measures for architecture evolution. |

## Источники из приложенного архива

| ID | Файл в архиве | Для чего использован |
|----|---------------|----------------------|
| A1 | `Other/AI-First-Repositories.md` | Идея AI-first repository, context window challenge, repository as AI-collaboration surface. |
| A2 | `Other/RepoGraph.md` | Repository-level code graph, навигация по кодовой базе для AI software engineering. |
| A3 | `ArchitectureAsCode/partitioned/*` | Structural alignment, architecture constraints, fitness functions, architecture/code metrics. |
| A4 | `ai-first-approaches/02-context-engineering.md` | Tiered context, progressive disclosure, context transformations. |
| A5 | `ai-first-approaches/03-agent-driven-development.md` | Spec-driven / agent-driven development, проектная документация как рабочий артефакт. |
| A6 | `ai-first-approaches/04-observability-and-evals.md` | Evals, traces/logs/metrics, application legibility. |
| A7 | `ai-first-approaches/05-knowledge-and-memory.md` | AGENTS.md как навигация, repository as knowledge system. |
| A8 | `ai-first-approaches/06-codebase-as-prompt.md` | Codebase as prompt, deep modules, sinks/not pipes, AI-friendly code. |
| A9 | `ai-first-approaches/08-security-and-governance.md` | HITL, sandboxing, governance, permission boundaries. |
| A10 | `book/10_...агент_не_чат.md` | Различие agent vs chat, tool use, external state. |
| A11 | `book/12_RAG.md` | RAG anatomy, chunking, retrieval, context stuffing limits. |
| A12 | `book/13_...антигаллюцинационный_контур.md` | Generator/verifier, anti-hallucination loop. |
| A13 | `book/14_...оценка_качества_llm_систем.md` | Golden datasets, evals as contracts, regression checks. |
| A14 | `book/15_...безопасность_llm_систем.md` | LLM-specific security risks and mitigations. |
| A15 | `book/16_...архитектура_кода.md` | AI-friendly code architecture: modules, contracts, explicit structure. |
| A16 | `book/17_...наблюдаемость_и_эксплуатация.md` | OTel GenAI, traces, runtime metrics, AI system operations. |
| A17 | `check_repo_methodics_old/*` | Предыдущая версия методики, структура score/evidence, audit checklist. |

## Как читать ссылки в методике

- Ссылки вида `[E7]` — внешний источник из таблицы выше.
- Ссылки вида `[A12]` — источник из приложенного архива.
- Источник не означает дословное заимствование: большинство пунктов ниже — синтез архивных материалов, актуальных docs/standards и практической адаптации под Python-репозиторий 50–100 KLOC.
