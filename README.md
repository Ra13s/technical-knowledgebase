# Technical Knowledgebase

A small, **actionable** engineering knowledgebase for things we can reuse while building software: APIs, annotations, libraries, configuration, implementation recipes, tests/checks, coding-agent workflows and concrete decision rules.

The admission test is simple:

> **Could we reasonably use this in a real development task? What exactly would we do differently?**

If the answer is vague, the item belongs in the processed-source log, not canonical knowledge.

## Canonical knowledge

### Java / persistence

- [`Hibernate @Immutable`](topics/java/hibernate-immutable.md) — make an entity/attribute/collection intentionally read-only to Hibernate and avoid entity dirty checking.

### Spring / AI

- [`Spring AI ToolSearchToolCallingAdvisor`](topics/spring/spring-ai-tool-search-advisor.md) — progressively disclose tools instead of injecting a large tool catalog into every model request.

### Coding agents

- [`Evidence-gated coding-agent edits`](topics/ai/evidence-gated-coding-agent-edits.md) — require observable evidence before bug-fix agents edit code or submit a patch, and make “no code change required” a valid successful outcome.

## Intake

- [`sources/processed.md`](sources/processed.md) — deduplication ledger and dispositions.
- [`intakes/`](intakes/) — short records of what was added, rejected and worth experimenting with.
- [`CONTRIBUTING.md`](CONTRIBUTING.md) — canonical admission and entry rules.
