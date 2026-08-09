# Agentic software development

## Current understanding

Coding agents change the limiting resource in software delivery. Code generation can be parallelized cheaply; specification quality, architecture, verification, integration and human attention do not scale at the same rate.

The useful unit of optimization is therefore not "lines of code generated per hour" but **verified, integrated change per unit of human attention**.

## External evidence

- Rachel Laycock's "Conductor Developer" describes developers coordinating multiple concurrent agents and argues that human attention becomes the scarce resource as code generation scales.
- Scott Logic's Java-to-Rust migration work found that agents can satisfy a nominal requirement with architecturally poor implementations. Their response was to invest in traceability, mutation testing, stronger specifications and small verifiable increments rather than relying on subjective trust in generated code.
- JetBrains exposing IntelliJ Java/Kotlin intelligence through LSP is evidence that coding agents increasingly benefit from semantic program tooling rather than treating repositories as collections of text files.
- Embabel 1.0 and the Jakarta Agentic AI milestone are evidence that agent orchestration is moving toward explicit actions, goals, state and lifecycle abstractions.

## Our position

1. Optimize agent workflows for **verification throughput**, not generation throughput.
2. Parallel agents work best on tasks with clear ownership boundaries and independently verifiable outputs.
3. Give agents executable feedback loops: tests, type checking, static analysis, build commands and domain-specific checks.
4. Prefer semantic repository tools (compiler/IDE/LSP/indexes) over raw text search when available.
5. Keep important architecture and product intent in versioned, machine-readable repository knowledge so agents can retrieve it repeatedly.
6. Increasing agent autonomy should be conditional on stronger confidence mechanisms, not on model capability alone.

## Confidence and limitations

**Confidence: medium-high.** Multiple independent sources point in the same direction, but practices are changing quickly and there is not yet a stable industry optimum for agent count, review topology or autonomy level.

The optimal workflow depends heavily on task independence, repository modularity, test quality and the cost of a wrong change.

## Counter-evidence / tensions

- Better models may reduce supervision requirements, so today's human-attention bottleneck may move.
- Too much process around agents can erase the productivity benefit. Confidence mechanisms should be automated wherever possible.
- Parallelism can increase merge conflicts and duplicated investigation when task decomposition is poor.

## Practical implications

Treat agent orchestration as an engineering system with explicit inputs, boundaries, checks and observability. A repository intended for agent-heavy development should make common validation operations obvious and cheap to execute.

## Experiments

### Parallel-agent saturation

Run comparable batches with 1, 3, 5 and 10 independent agents. Measure:

- accepted changes
- human review minutes
- rework/rejection rate
- merge conflicts
- escaped defects
- token/compute cost
- elapsed time to integrated result

The useful result is the point where additional agent concurrency stops improving verified throughput.

### Semantic tooling

Compare the same repository task using text-only navigation versus IDE/LSP semantic navigation. Measure tokens, elapsed time, incorrect symbol assumptions and resulting defects.

## Open questions

- What is the best practical metric for human attention consumed by agent work?
- When should verification itself be delegated to independent agents?
- How much repository knowledge should be always-loaded versus retrieved on demand?
- What task granularity maximizes independence without losing architectural coherence?

## Sources

- https://martinfowler.com/rachels-ramblings/conductor-developer.html
- https://blog.scottlogic.com/2026/08/04/ai-migration-techniques.html
- https://blog.jetbrains.com/idea/2026/08/intellij-idea-goes-lsp/
- https://www.infoq.com/news/2026/08/embabel-1/
- https://foojay.io/today/jakarta-agentic-ai-hits-its-first-milestone/
