# Technical Knowledgebase

A small, **actionable** engineering knowledgebase for things we can reuse while building software: APIs, annotations, libraries, configuration, implementation recipes, tests/checks, coding-agent workflows and concrete decision rules.

The admission test is simple:

> **Could we reasonably use this in a real development task? What exactly would we do differently?**

If the answer is vague, the item belongs in the processed-source log, not canonical knowledge.

## Clone this system

The intake workflow itself is reusable:

- [`prompts/knowledgebase-intake.md`](prompts/knowledgebase-intake.md) — the full governing prompt used to discover, admit, synthesize, publish and merge new knowledge.
- [`docs/chatgpt-setup.md`](docs/chatgpt-setup.md) — how to fork/clone the repository, connect GitHub to ChatGPT, test a manual run, and run the intake repeatedly with ChatGPT Scheduled Tasks.

The prompt lives in Git deliberately. A scheduled ChatGPT task can stay small and load the current prompt from the repository at the start of every run, so improvements to the system are versioned and automatically picked up later.

## Canonical knowledge

### Java

- [`Switching over evolving sealed APIs`](topics/java/evolving-sealed-api-switches.md) — keep compiler exhaustiveness for hierarchies we own, but control the runtime failure path for independently evolving sealed dependencies.

### Persistence

- [`Hibernate @Immutable`](topics/java/hibernate-immutable.md) — make an entity/attribute/collection intentionally read-only to Hibernate; for immutable-by-convention JSON/custom values, avoid unnecessary deep-copy/cache serialization with a compatible `@Mutability` plan.
- [`Hibernate @EmbeddedTable`](topics/java/hibernate-embedded-table.md) — map a complete embeddable to a secondary table without repetitive per-member table overrides in Hibernate ORM 7.2+.
- [`Hibernate ORM 7.4 core @Audited`](topics/java/hibernate-core-audited.md) — keep entity audit history in Hibernate core, query historical state through normal sessions/HQL, and optionally migrate a compatible Envers schema.

### Spring / Java

- [`Spring Boot outbound SSRF mitigation with InetAddressFilter`](topics/spring/spring-boot-inetaddressfilter-ssrf.md) — filter resolved outbound HTTP destinations globally or per client, with explicit CIDR policies and tests.
- [`Spring Data type-safe property paths`](topics/spring/spring-data-typed-property-paths.md) — replace compile-time-known string property names with refactoring-safe method references and typed nested paths.
- [`JSpecify + NullAway null-safety`](topics/spring/jspecify-null-safety.md) — make packages non-null by default with `@NullMarked`, mark nullable type uses explicitly and optionally fail the build on contract violations.
- [`DuckDB for set-based Spring Batch transforms`](topics/spring/spring-batch-duckdb-transforms.md) — replace large in-memory grouping/join/aggregation loops with an embedded analytical SQL step when the workload fits.

### Spring / AI

- [`Spring AI HyDE query transformation`](topics/spring/spring-ai-hyde-retrieval.md) — turn conversational questions into answer-like retrieval queries when user vocabulary does not match indexed documentation; keep the hypothetical text out of answer evidence.
- [`Spring AI structured output validation and self-correction`](topics/spring/spring-ai-structured-output-validation.md) — combine provider-native schema enforcement with bounded response validation/retry for typed LLM outputs.
- [`Spring AI ToolSearchToolCallingAdvisor`](topics/spring/spring-ai-tool-search-advisor.md) — progressively disclose tools instead of injecting a large tool catalog into every model request.

### Coding agents / AI protocols

- [`Evidence-gated coding-agent edits`](topics/ai/evidence-gated-coding-agent-edits.md) — require observable evidence before edits, include authoritative external state when code is not the source of truth, isolate parallel edits, and gate PR submission on deterministic checks.
- [`Narrow-contract background coding agent`](topics/ai/narrow-contract-background-coding-agent.md) — run background agents only from explicit triggers, reject underspecified tasks before editing, use a fresh-context reviewer, and preserve normal human/CI/security gates.
- [`Deterministic outer loop for coding-agent platforms`](topics/ai/deterministic-outer-loop-agent-platform.md) — keep snapshots, validation, retries, lifecycle hooks and publication under deterministic workflow control while the model handles reasoning and edits.
- [`Ablate coding-agent harness scaffolding when models change`](topics/ai/ablate-agent-harness-scaffolding.md) — re-test planners, context resets, evaluators and other non-safety scaffolding one component at a time after model upgrades instead of preserving obsolete workarounds as cargo cult.
- [`Search-based code optimization agents with hard fitness gates`](topics/ai/search-based-code-optimization-agents.md) — use LLMs to generate candidates inside a bounded search loop while deterministic correctness checks, benchmarks, diagnostics and budgets decide what survives.
- [`Layered scoped agent memory with stateless reasoning sessions`](topics/ai/layered-scoped-agent-memory.md) — keep durable memory external, scoped and versionable; consolidate recent observations, retrieve only relevant memory, and reconstruct each reasoning session from live source-of-truth state.
- [`Precision-first multi-stage AI code review`](topics/ai/precision-first-multi-stage-code-review.md) — combine specialized reviewers, independent grading, semantic dedupe and production feedback to optimize review precision rather than comment volume.
- [`Bind agent approvals to the exact side effect`](topics/ai/enforcement-bound-agent-approvals.md) — require mandatory human approval at the execution boundary, bind it to exact arguments/target/identity/expiry, and revalidate before executing.
- [`Review AI-generated Maven build changes`](topics/ai/review-ai-generated-maven-build-changes.md) — inspect dependency tree, effective POM/settings and plugin dependencies whenever an agent changes Maven build configuration.
- [`OpenAI Programmatic Tool Calling`](topics/ai/openai-programmatic-tool-calling.md) — use generated code for bounded deterministic tool-call reduction while keeping approval-, semantic- and citation-sensitive work direct.
- [`MCP protocol upgrades and conformance`](topics/ai/mcp-protocol-upgrades-and-conformance.md) — isolate protocol-version adapters at the boundary, test old/new contracts and run the official MCP conformance suite in CI.

### Testing / API contracts

- [`Microcks Testcontainers contract testing`](topics/testing/microcks-testcontainers-contract-tests.md) — generate mocks and provider conformance tests from the same OpenAPI/AsyncAPI artifact inside local JUnit/CI tests.

### Platform / CI / dependency automation

- [`One aggregate required check for conditional GitHub Actions CI`](topics/platform/github-actions-aggregate-required-check.md) — keep path-specific CI conditional while exposing one always-present status check to branch protection and merge queues.
- [`Renovate + Gradle dependency verification metadata`](topics/platform/renovate-gradle-verification-metadata.md) — regenerate Gradle verification metadata in the same Renovate dependency-update PR with tightly allowlisted post-upgrade commands.

## Intake

- [`sources/processed.md`](sources/processed.md) — legacy deduplication ledger.
- [`sources/processed/`](sources/processed/) — append-only dated processed-source logs for newer runs.
- [`intakes/`](intakes/) — editorial run syntheses: what changed in our engineering model, what was added/rejected, and experiments worth trying.
- [`CONTRIBUTING.md`](CONTRIBUTING.md) — canonical admission and entry rules.
