# Processed sources

This is the deduplication ledger for intake runs. A source listed here or in an open intake PR is treated as processed.

| Processed | Source | Title | URL | Disposition |
|---|---|---|---|---|
| 2026-08-09 | Baeldung Java Weekly | Java Weekly #658 | https://www.baeldung.com/java-weekly-658 | incorporated — discovery source; followed high-value links |
| 2026-08-09 | InfoQ | Virtual Threads after JDK 24 | https://www.infoq.com/articles/virtual-threads-after-jdk24/ | incorporated — resource bounding and ThreadLocal implications |
| 2026-08-09 | Martin Fowler | The economic benefits of refactoring for AI | https://martinfowler.com/articles/exploring-gen-ai/refactoring-economic-benefit.html | incorporated — modularity reduces agent context cost |
| 2026-08-09 | Martin Fowler / Rachel Laycock | The Conductor Developer | https://martinfowler.com/rachels-ramblings/conductor-developer.html | incorporated — human attention becomes agent throughput constraint |
| 2026-08-09 | JetBrains | IntelliJ IDEA goes LSP | https://blog.jetbrains.com/idea/2026/08/intellij-idea-goes-lsp/ | evidence-only — semantic tooling for coding agents |
| 2026-08-09 | InfoQ | Embabel 1.0 | https://www.infoq.com/news/2026/08/embabel-1/ | evidence-only — planning-oriented agent orchestration |
| 2026-08-09 | Foojay | Jakarta Agentic AI first milestone | https://foojay.io/today/jakarta-agentic-ai-hits-its-first-milestone/ | evidence-only — Java agent abstractions are beginning to standardize |
| 2026-08-09 | Scott Logic | AI migration techniques | https://blog.scottlogic.com/2026/08/04/ai-migration-techniques.html | incorporated — verification and confidence mechanisms |
| 2026-08-09 | All Things Distributed | On building scalable control planes | https://www.allthingsdistributed.com/2026/08/on-building-scalable-control-planes.html | incorporated — reconciliation and static stability |
| 2026-08-09 | Christian Posta | Credential brokering patterns for AI agent egress, part 2 | https://blog.christianposta.com/credential-brokering-patterns-for-ai-agent-egress-part2/ | evidence-only — delegated identity boundary |
| 2026-08-09 | Event-Driven.io | Fixing bugs in Event Sourcing is hard | https://event-driven.io/en/fixing-bugs-in-event-sourcing-is-hard/ | evidence-only — provenance enables targeted corrective events |
| 2026-08-09 | Foojay | Idempotent Spring Boot Starter | https://foojay.io/today/idempotent-spring-boot-starter/ | evidence-only — idempotency is not exactly-once by default |
| 2026-08-09 | Quarkus | Faster, leaner, predictable Quarkus | https://quarkus.io/blog/quarkus-insights-255-faster-leaner-predictable-quarkus/ | skipped for canonical guidance — useful benchmark but workload-specific |
| 2026-08-13 | Hibernate ORM 7.1 | `org.hibernate.annotations.Immutable` Javadoc | https://docs.hibernate.org/orm/7.1/javadocs/org/hibernate/annotations/Immutable.html | incorporated — concrete ORM annotation and behavior |
| 2026-08-13 | Spring AI | Tool Search Tool / `ToolSearchToolCallingAdvisor` | https://docs.spring.io/spring-ai/reference/2.0-SNAPSHOT/api/tools/tool-search-tool.html | incorporated — concrete dynamic tool discovery setup |
| 2026-08-13 | Spring | Tool Calling in Spring AI 2.0 | https://spring.io/blog/2026/06/15/spring-ai-composable-tool-calling/ | evidence-for-existing-entry — setup and version details for tool search |
| 2026-08-13 | Gloaguen et al. | Coding Agents Don't Know When to Act | https://arxiv.org/abs/2605.07769 | incorporated — reproduce/abstain rule for bug-fix agents |
| 2026-08-13 | Xu et al. | Preventing Premature Commitment in Coding Agents with an Evidence-Conditioned Execution Layer | https://arxiv.org/abs/2607.28815 | incorporated — structured evidence gate before edits/submission |
| 2026-08-17 | Hibernate ORM | `@EmbeddedTable` Javadoc | https://docs.hibernate.org/orm/7.2/javadocs/org/hibernate/annotations/EmbeddedTable.html | incorporated — concrete secondary-table mapping annotation |
| 2026-08-17 | Hibernate ORM | Releases and compatibility matrix | https://hibernate.org/orm/releases/ | evidence-for-existing-entry — version/Spring Boot compatibility for `@EmbeddedTable` |
| 2026-08-17 | Oracle Java SE 26 | JLS Chapter 13: Binary Compatibility | https://docs.oracle.com/en/java/javase/26/docs/specs/jls/jls-13.html | incorporated — concrete sealed-API switch compatibility rule |
| 2026-08-17 | Oracle Java SE 26 | `java.lang.MatchException` | https://docs.oracle.com/en/java/javase/26/docs/api/java.base/java/lang/MatchException.html | evidence-for-existing-entry — runtime failure mode for stale exhaustive switches |
| 2026-08-17 | Model Context Protocol | The 2026-07-28 Specification | https://blog.modelcontextprotocol.io/posts/2026-07-28/ | incorporated — concrete request-contract changes for MCP migrations |
| 2026-08-17 | Inside Java | Evolving a Java MCP Server During MCP Specification Upgrades | https://inside.java/2026/08/12/java-mcp-migration/ | incorporated — compatibility adapter and dual-path test recipe |
| 2026-08-17 | Model Context Protocol | MCP Conformance Test Framework | https://github.com/modelcontextprotocol/conformance | incorporated — executable client/server conformance checks |
| 2026-08-17 | OpenAI | How agents are transforming work | https://openai.com/index/how-agents-are-transforming-work/ | skipped-not-actionable — adoption/task-horizon evidence without a reusable implementation technique |
| 2026-08-17 | OpenJDK | Convenience Methods for JSON Documents (draft JEP) | https://openjdk.org/jeps/8344154 | skipped-low-confidence — draft/incubating API; no production recommendation yet |
| 2026-08-21 | Baeldung Java Weekly | Java Weekly #659 | https://www.baeldung.com/java-weekly-659 | incorporated — discovery source; followed actionable linked material and skipped already-processed MCP migration |
| 2026-08-21 | Foojay | DuckDB in Spring Batch: Replace In-Memory Java Loops with One SQL Statement | https://foojay.io/today/duckdb-in-spring-batch-replace-in-memory-java-loops-with-one-sql-statement/ | incorporated — concrete set-based batch transform recipe |
| 2026-08-21 | DuckDB | Java JDBC Client | https://duckdb.org/docs/lts/clients/java | evidence-for-existing-entry — JDBC setup and supported client behavior |
| 2026-08-21 | Spring Batch | `Tasklet` / `StepBuilder.tasklet` reference | https://docs.spring.io/spring-batch/reference/api/org/springframework/batch/core/step/tasklet/Tasklet.html | evidence-for-existing-entry — correct non-item-oriented Spring Batch step shape |
| 2026-08-21 | Marc Philipp | One required check to rule them all | https://marcphilipp.de/blog/2026/08/10/one-required-check-to-rule-them-all/ | incorporated — aggregate required-check pattern for conditional GitHub Actions CI |
| 2026-08-21 | GitHub Docs | Troubleshooting required status checks | https://docs.github.com/en/pull-requests/how-tos/merge-and-close-pull-requests/troubleshooting-required-status-checks | evidence-for-existing-entry — confirms path-filtered required workflow can remain pending |
| 2026-08-21 | GitHub Docs | Reusing workflow configurations | https://docs.github.com/en/actions/reference/workflows-and-actions/reusing-workflow-configurations | evidence-for-existing-entry — `workflow_call` and reusable-workflow constraints |
| 2026-08-21 | GitHub Docs | `merge_group` event | https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#merge_group | evidence-for-existing-entry — merge-queue required checks must trigger on `merge_group` |
| 2026-08-21 | Spring | This Week in Spring — August 18th, 2026 | https://spring.io/blog/2026/08/18/this-week-in-spring-august-18-2026/ | incorporated — discovery source; followed JSpecify/null-safety guidance |
| 2026-08-21 | Spring Framework | Null-safety reference | https://docs.spring.io/spring-framework/reference/core/null-safety.html | incorporated — JSpecify package/type-use recipe and recommended NullAway settings |
| 2026-08-21 | Spring | Null-safe applications with Spring Boot 4 | https://spring.io/blog/2025/11/12/null-safe-applications-with-spring-boot-4/ | evidence-for-existing-entry — Boot 4 portfolio coverage and javac compatibility guidance |
| 2026-08-21 | NullAway | Configuration | https://github.com/uber/NullAway/wiki/Configuration | evidence-for-existing-entry — `OnlyNullMarked`, contracts and JSpecify flags |
| 2026-08-21 | NullAway | JSpecify Support | https://github.com/uber/NullAway/wiki/JSpecify-Support | evidence-for-existing-entry — JSpecify mode maturity and JDK/compiler constraints |
| 2026-08-21 | Martin Fowler / Birgitta Böckeler | TDD inside the agent loop — theater or actual value? | https://martinfowler.com/articles/exploring-gen-ai/tdd-in-the-agent-loop.html | skipped-low-confidence — actionable hypothesis but exploratory sample is too small for canonical agent policy; retained as experiment idea |
| 2026-08-21 | JetBrains | How to Use AI Agents in IntelliJ IDEA With ACP | https://blog.jetbrains.com/idea/2026/08/how-to-use-ai-agents-in-intellij-idea-with-acp/ | skipped-lower-priority — concrete custom-agent setup, but narrower expected reuse than this run's selected entries |
| 2026-08-21 | JetBrains AI Assistant | Agent Client Protocol (ACP) | https://www.jetbrains.com/help/ai-assistant/acp.html | skipped-lower-priority — primary setup details for the deferred ACP candidate |
| 2026-08-21 | JetBrains AI Assistant | Skills | https://www.jetbrains.com/help/ai-assistant/agent-skills.html | skipped-lower-priority — concrete Codex/Claude skill locations; defer until agent-skill management becomes a recurring need |
