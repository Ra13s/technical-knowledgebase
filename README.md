# Technical Knowledgebase

A small, **actionable** engineering knowledgebase for things we can reuse while building software: APIs, annotations, libraries, configuration, implementation recipes, tests/checks, coding-agent workflows and concrete decision rules.

The admission test is simple:

> **Could we reasonably use this in a real development task? What exactly would we do differently?**

If the answer is vague, the item belongs in the processed-source log, not canonical knowledge.

## Canonical knowledge

### Java

- [`Switching over evolving sealed APIs`](topics/java/evolving-sealed-api-switches.md) — keep compiler exhaustiveness for hierarchies we own, but control the runtime failure path for independently evolving sealed dependencies.

### Persistence

- [`Hibernate @Immutable`](topics/java/hibernate-immutable.md) — make an entity/attribute/collection intentionally read-only to Hibernate and avoid entity dirty checking.
- [`Hibernate @EmbeddedTable`](topics/java/hibernate-embedded-table.md) — map a complete embeddable to a secondary table without repetitive per-member table overrides in Hibernate ORM 7.2+.

### Spring / Java

- [`JSpecify + NullAway null-safety`](topics/spring/jspecify-null-safety.md) — make packages non-null by default with `@NullMarked`, mark nullable type uses explicitly and optionally fail the build on contract violations.
- [`DuckDB for set-based Spring Batch transforms`](topics/spring/spring-batch-duckdb-transforms.md) — replace large in-memory grouping/join/aggregation loops with an embedded analytical SQL step when the workload fits.

### Spring / AI

- [`Spring AI ToolSearchToolCallingAdvisor`](topics/spring/spring-ai-tool-search-advisor.md) — progressively disclose tools instead of injecting a large tool catalog into every model request.

### Coding agents / AI protocols

- [`Evidence-gated coding-agent edits`](topics/ai/evidence-gated-coding-agent-edits.md) — require observable evidence before bug-fix agents edit code or submit a patch, and make “no code change required” a valid successful outcome.
- [`MCP protocol upgrades and conformance`](topics/ai/mcp-protocol-upgrades-and-conformance.md) — isolate protocol-version adapters at the boundary, test old/new contracts and run the official MCP conformance suite in CI.

### Platform / CI

- [`One aggregate required check for conditional GitHub Actions CI`](topics/platform/github-actions-aggregate-required-check.md) — keep path-specific CI conditional while exposing one always-present status check to branch protection and merge queues.

## Intake

- [`sources/processed.md`](sources/processed.md) — deduplication ledger and dispositions.
- [`intakes/`](intakes/) — short records of what was added, rejected and worth experimenting with.
- [`CONTRIBUTING.md`](CONTRIBUTING.md) — canonical admission and entry rules.
