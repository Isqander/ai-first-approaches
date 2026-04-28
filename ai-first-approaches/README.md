# AI-First подходы — справочник

> Структурированное собрание методик, подходов и архитектурных решений для построения AI-агентных систем. Актуально на 2026 год.
> Не привязан жёстко к AI OS — это knowledge base, полезный для любого проекта с AI-агентами.

---

## Содержание

| # | Файл | Тема | Кратко |
|---|------|------|--------|
| 01 | [01-multi-agent-architecture](01-multi-agent-architecture.md) | Мультиагентная архитектура | Паттерны оркестрации, роли, fleet coordination, competition mode |
| 02 | [02-context-engineering](02-context-engineering.md) | Context engineering | Управление контекстом как инженерная дисциплина: tiered model, компактизация, caching |
| 03 | [03-agent-driven-development](03-agent-driven-development.md) | Agent-Driven Development | AIDD, spec-driven, subagent-driven, plan mode, greedy reading |
| 04 | [04-observability-and-evals](04-observability-and-evals.md) | Наблюдаемость и оценка | Трейсы, логирование, eval-driven development, LLM-as-a-judge |
| 05 | [05-knowledge-and-memory](05-knowledge-and-memory.md) | Знания и память | AGENTS.md, progressive disclosure, knowledge graphs, doc-gardening |
| 06 | [06-codebase-as-prompt](06-codebase-as-prompt.md) | Кодовая база как промпт | Deep modules, sinks not pipes, bounded contexts, autonomous feedback loop |
| 07 | [07-self-modification](07-self-modification.md) | Самомодификация | Evolution, garbage collection, self-created tools, golden principles |
| 08 | [08-security-and-governance](08-security-and-governance.md) | Безопасность и governance | HITL, sandboxing, trust levels, policy enforcement |

---

## Как пользоваться

**Для проектирования системы** → начни с 01 (архитектура) и 08 (безопасность).
**Для организации разработки** → 03 (AIDD) и 06 (codebase as prompt).
**Для настройки наблюдаемости** → 04 (observability) и 05 (knowledge).
**Для эволюции системы** → 07 (self-modification).

Каждый файл — самодостаточный. Перекрёстные ссылки указывают на связанные разделы.

---

## Источники

| Документ/подход | Автор/организация | Что взяли |
|----------------|-------------------|-----------|
| Harness Engineering | OpenAI | Humans steer / agents execute, AGENTS.md как TOC, entropy/GC |
| AIDD Methodology | — | Developer=architect, LLM=implementer, Context First |
| Scaling Agents | Cursor | Planner-Worker-Judge, модель > промпт > инфра |
| Context Engineering (ADK) | Google | Tiered context model, processors, scoped handoffs |
| Sinks Not Pipes | — | Deep modules, contained effects, progressive disclosure |
| Background Agents | — | Fleet coordination, event-driven, sandboxed execution |
| Demystifying Evals | Anthropic | Eval harness, pass@k/pass^k, capability vs regression |
| LLM Metrics | — | Graders, LLM-as-a-judge, eval-driven development |
| Autonomous Feedback Loop | — | DEBUG.md, verify before document, hot-eval first |
| Superpowers | — | Composable skills: brainstorm → spec → plan → execute |
| GRACE | — | Contract-driven + semantic markup + knowledge graphs |
| 12 Rules of LLM Dev | — | 4 stages, metrics first, smaller model when possible |
| Claude Code Shorthand | — | Skills, hooks, subagents, MCP, context management |
| Cursor AI Engineering | — | Bounded contexts, invariants, pre-prompt policy |
| Talks on Graphs | — | Code Property Graphs, SCIP, call graphs for LLM context |
| HardFork Conference | — | Layers model, multi-model routing, MCP as USB-C |
