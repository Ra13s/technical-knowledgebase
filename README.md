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

## Run briefs

A compact human-readable record of what each intake changed in our engineering model. Full source-by-source synthesis remains under [`intakes/`](intakes/).

### 2026-09-30 — Profile the work before optimizing it

**Main idea:** once an agent system is working, optimize the bottleneck revealed by cross-run evidence rather than the component that is easiest to measure. Cursor, AgentPProf and Augment all point toward the same operating loop: profile repeated work, find the dominant token/time/queue hotspot, change one thing, and re-measure on the same workload.

**Added:** semantic trajectory profiling for agent fleets, JDK 27 JFR in-process redaction, and Kubernetes 1.37 HPA scale-to-zero. **Improved:** agent-fleet economics now includes cache-shaped request design, progressive tool disclosure, subagent coordination cost, and queue growth as workflow inventory.

**Try next:** build a semantic flamegraph over 50–100 frozen agent sessions, A/B a cache-shaped request layout, and add queue-age metrics across task → review → CI → verification.

[Read the full 2026-09-30 intake →](intakes/2026-09-30.md)

<details>
<summary><strong>2026-09-26 — Put guarantees in infrastructure, not in “be careful” prompts</strong></summary>

**Main idea:** retry safety, execution evidence and authorization-grade state are system-boundary properties. The reasoning plane can stay probabilistic; the effect plane needs stable operation identity and idempotency; the evidence plane needs authoritative telemetry outside the agent's writable workspace.

**Added:** idempotent side-effect contracts for agent tools, external authoritative agent telemetry, and Spring AI TypeSafe Jev decision gates. The useful boundary is simple: if duplicate effects matter, fix the tool contract; if auditability matters, record outside the task workspace; if the result is only a branch/score/classification, consider a typed decision primitive before another generative call.

**Try next:** inject lost-ack/late-commit/redelivery faults into one mutating tool, reconstruct an unattended run from control-plane telemetry alone, and benchmark a Jev cascade against a current chat-model classifier.

[Read the full 2026-09-26 intake →](intakes/2026-09-26.md)
</details>

<details>
<summary><strong>2026-09-22 — Every autonomous claim needs identity, proof and lifecycle state</strong></summary>

**Main idea:** “the model found it”, “the config changed” or “the request succeeded” are too weak when downstream state matters. High-autonomy systems should attach explicit identity, independent evidence and observable completion to important claims and transitions.

**Added:** proof-gated agentic security review, workload-class capacity isolation with bounded recovery, and Kubernetes `StorageVersionMigration`. Google Mantis showed discovery should be separated from trust; Datadog showed recovery itself can become shared-system load; Kubernetes made hidden stored-object rewrites first-class and observable.

**Try next:** compare generic vs proof-gated security review, chaos-test dual retry budgets across many concurrent agents, and verify workload identity survives every model/tool/queue/CI hop.

[Read the full 2026-09-22 intake →](intakes/2026-09-22.md)
</details>

<details>
<summary><strong>2026-09-18 — Turn organizational knowledge into executable capabilities</strong></summary>

**Main idea:** repeatable expert knowledge becomes far more reusable when it is packaged as a versioned playbook with explicit tools, permissions, validation and outputs instead of being pasted into ever-larger prompts. LinkedIn CAPT, DoorDash Flux and Spring AI Agent Skills independently converge on progressive discovery plus deterministic capability enforcement.

**Added:** executable agent playbooks, Java 26 HTTP/3 via `HttpClient`, and a Kubernetes DRA migration bridge that preserves existing workload contracts. **Improved:** search-based optimization now requires proving the benchmark resembles production and freezing the oracle before the agent starts optimizing against it.

**Try next:** encode one real internal procedure as a playbook, pilot a repository-local Spring `SkillsTool`, canary HTTP/3 and record the negotiated protocol, or migrate one accelerator pool to DRA without touching workload manifests.

[Read the full 2026-09-18 intake →](intakes/2026-09-18.md)
</details>

## Canonical knowledge

### Java

- [`Switching over evolving sealed APIs`](topics/java/evolving-sealed-api-switches.md) — keep compiler exhaustiveness for hierarchies we own, but control the runtime failure path for independently evolving sealed dependencies.
- [`Java 26 HTTP/3 with java.net.http.HttpClient`](topics/java/java-26-http3-httpclient.md) — opt into HTTP/3 with the standard client, choose fallback/discovery behavior explicitly and verify the negotiated response version on the real network path.
- [`Java 27 JFR in-process redaction`](topics/java/java-27-jfr-in-process-redaction.md) — keep selected sensitive startup data out of supported JFR events at capture time, preserve the runtime defaults and canary-test the exact coverage.

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
- [`Spring AI TypeSafe Jev decision gates`](topics/spring/spring-ai-typesafe-jev-decision-gates.md) — use typed judgements for routing, evaluation and bounded refinement when the desired output is a decision rather than generated prose.

### Coding agents / AI protocols

- [`Evidence-gated coding-agent edits`](topics/ai/evidence-gated-coding-agent-edits.md) — require observable current-state evidence before edits, bind CI/tests to the exact head SHA, and verify the authoritative postcondition after merge.
- [`Narrow-contract background coding agent`](topics/ai/narrow-contract-background-coding-agent.md) — run background agents only from explicit triggers, reject underspecified tasks before editing, use a fresh-context reviewer, and preserve normal human/CI/security gates.
- [`Deterministic outer loop for coding-agent platforms`](topics/ai/deterministic-outer-loop-agent-platform.md) — keep snapshots, validation, retries, lifecycle hooks and publication under deterministic workflow control while the model handles reasoning and edits.
- [`Expose one agent harness through a stable event protocol`](topics/ai/stable-agent-harness-event-protocol.md) — keep thread/turn/item lifecycle, approvals and durable task state in one harness and expose it through a versioned bidirectional protocol to IDE, CLI, web and remote clients.
- [`Build agent-addressable application feedback loops`](topics/ai/agent-addressable-application-feedback-loop.md) — give coding agents typed inspect/navigate/act control surfaces for running apps so UI validation is not bottlenecked on screenshots, simulator clicks or human narration.
- [`Encode organizational workflows as executable agent playbooks`](topics/ai/executable-agent-playbooks.md) — package repeatable engineering work as discoverable, versioned workflows with explicit tools, permissions, deterministic gates and outputs rather than burying institutional knowledge in static prompts/docs.
- [`Ablate coding-agent harness scaffolding when models change`](topics/ai/ablate-agent-harness-scaffolding.md) — re-test planners, context resets, evaluators and other non-safety scaffolding one component at a time after model upgrades instead of preserving obsolete workarounds as cargo cult.
- [`Search-based code optimization agents with hard fitness gates`](topics/ai/search-based-code-optimization-agents.md) — use LLMs to generate candidates inside a bounded search loop; validate correctness first and, for production code, ground/freeze the fitness benchmark against real workload signals before optimizing.
- [`Layered scoped agent memory with stateless reasoning sessions`](topics/ai/layered-scoped-agent-memory.md) — keep durable memory external, scoped and versionable; use live domain artifacts such as issues/PRs as handoff state where possible, and reconstruct each reasoning session from current source-of-truth state.
- [`Engineer agent fleets with outcome-denominated unit economics`](topics/ai/agent-fleet-unit-economics.md) — benchmark models on real work and optimize cost per accepted outcome, routing bounded work cheaply while measuring context/tool-call waste and quality.
- [`Profile agent fleets by semantic operation`](topics/ai/semantic-agent-trajectory-profiling.md) — aggregate many trajectories into semantic token/time/effect profiles so harness optimization targets repeated fleet hotspots rather than anecdotal single runs.
- [`Protect shared agent platforms with workload classes and bounded recovery`](topics/ai/agent-platform-capacity-isolation.md) — inventory full agent trajectories, propagate workload identity, reserve capacity by service class and bound recovery per run and across shared dependencies.
- [`Gate coding-agent harness configuration as a supply chain`](topics/ai/agent-harness-supply-chain-gates.md) — lint persistent agent configuration at install and assembly boundaries; pin runtime tool dependencies and gate only validated, deterministic defect classes.
- [`Resolve repository instructions by target path`](topics/ai/path-scoped-repository-instructions.md) — compose only ancestor instructions relevant to the active path, refresh them as tools move through a monorepo, and bound/observe instruction context explicitly.
- [`Contain coding agents at the environment boundary first`](topics/ai/environment-first-agent-containment.md) — keep filesystem, network, process and credential blast radius deterministic; treat model safety and approval dialogs as additional layers rather than the final boundary.
- [`Precision-first multi-stage AI code review`](topics/ai/precision-first-multi-stage-code-review.md) — localize risk first, gather bounded primary evidence, use review-specific context, explicitly try to disprove candidate findings, then dedupe and publish only high-signal comments.
- [`Gate agentic security findings with localized threat models and proof`](topics/ai/agentic-security-presubmit-pipeline.md) — scan changes with live threat context, then require structural reachability or sandbox reproduction before surfacing findings or asking an agent to patch them.
- [`Bind agent approvals to the exact side effect`](topics/ai/enforcement-bound-agent-approvals.md) — require mandatory human approval at the execution boundary, bind it to exact arguments/target/identity/expiry, and revalidate before executing.
- [`Make agent side effects idempotent in the tool contract`](topics/ai/idempotent-agent-tool-side-effects.md) — attach stable logical operation keys to persistent writes, expose authoritative status, and avoid unsafe blind retries.
- [`Externalize authoritative agent telemetry`](topics/ai/external-authoritative-agent-telemetry.md) — keep model/tool/executor evidence in control-plane telemetry outside the mutable task workspace.
- [`Review AI-generated Maven build changes`](topics/ai/review-ai-generated-maven-build-changes.md) — inspect dependency tree, effective POM/settings and plugin dependencies whenever an agent changes Maven build configuration.
- [`OpenAI Programmatic Tool Calling`](topics/ai/openai-programmatic-tool-calling.md) — use generated code for bounded deterministic tool-call reduction while keeping approval-, semantic- and citation-sensitive work direct.
- [`MCP protocol upgrades and conformance`](topics/ai/mcp-protocol-upgrades-and-conformance.md) — isolate protocol-version adapters at the boundary, test old/new contracts and run the official MCP conformance suite in CI.

### Testing / API contracts

- [`Record/replay AI model calls with request-signed cassettes`](topics/testing/ai-model-vcr-record-replay-tests.md) — record realistic Spring AI/LangChain4j provider interactions once, then use strict request-matched playback for deterministic offline CI while keeping live evals separate.
- [`Microcks Testcontainers contract testing`](topics/testing/microcks-testcontainers-contract-tests.md) — generate mocks and provider conformance tests from the same OpenAPI/AsyncAPI artifact inside local JUnit/CI tests.

### Platform / CI / dependency automation

- [`Rootless Kubernetes nodes for agent and test sandboxes`](topics/platform/rootless-kubernetes-agent-sandbox.md) — run Kubernetes 1.37+ node components in a Linux user namespace for lower-privilege agent/integration-test clusters; validate CNI/CSI compatibility explicitly.
- [`Migrate Kubernetes extended resources to DRA without changing workloads`](topics/platform/kubernetes-dra-extended-resource-migration.md) — map existing resource names to DRA `DeviceClass` objects in Kubernetes 1.37+ so device allocation can migrate node-by-node without forcing workload manifests to adopt `ResourceClaim` immediately.
- [`Migrate stored Kubernetes API objects with StorageVersionMigration`](topics/platform/kubernetes-storage-version-migration.md) — declaratively rewrite CRD/API objects to the current storage version in Kubernetes 1.37+, with observable completion gates for API-version retirement and encryption-key rotation.
- [`Scale queue-driven workloads to zero with Kubernetes HPA 1.37`](topics/platform/kubernetes-hpa-scale-to-zero.md) — use external/object metrics that survive zero Pods, verify HPA ownership of the zero state, and measure cold-start plus metrics-pipeline risk.
- [`One aggregate required check for conditional GitHub Actions CI`](topics/platform/github-actions-aggregate-required-check.md) — keep path-specific CI conditional while exposing one always-present status check to branch protection and merge queues.
- [`Renovate + Gradle dependency verification metadata`](topics/platform/renovate-gradle-verification-metadata.md) — regenerate Gradle verification metadata in the same Renovate dependency-update PR with tightly allowlisted post-upgrade commands.

## Intake

- [`sources/processed.md`](sources/processed.md) — legacy deduplication ledger.
- [`sources/processed/`](sources/processed/) — append-only dated processed-source logs for newer runs.
- [`intakes/`](intakes/) — editorial run syntheses: what changed in our engineering model, what was added/rejected, and experiments worth trying.
- [`CONTRIBUTING.md`](CONTRIBUTING.md) — canonical admission and entry rules.
