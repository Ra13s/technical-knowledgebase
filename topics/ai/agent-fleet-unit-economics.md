# Engineer agent fleets with outcome-denominated unit economics

## What it is

A cost/quality operating model for coding-agent fleets: decompose spend into measurable drivers, benchmark models on real work, and optimize against **cost per useful outcome** rather than token price or request count alone.

Uber applies this across interactive and managed agents and measures outcomes such as cost per merged PR, review, alert triage or cleanup alongside quality signals such as F1, revert rate and MTTR.

## Use when

Use this when coding-agent usage is growing fast enough that aggregate spend is no longer actionable, or when model/tool changes need to be compared without confusing adoption growth with efficiency.

Typical triggers:

- multiple coding/review/on-call agents share one platform;
- model upgrades change both quality and price;
- tool schemas, long contexts or polling loops dominate token usage;
- teams need a defensible rule for routing work to cheaper models.

## Cost model

Decompose total spend instead of staring at one monthly number:

```text
spend =
  users
  × sessions / user
  × turns / session
  × requests / turn
  × tokens / request
  × price / token
```

Do not optimize the first two terms merely to make cost look good: successful adoption should often increase them. Concentrate on unnecessary model work in turns, requests and tokens, while measuring quality separately.

## Implementation workflow

### 1. Define an outcome metric per managed agent

Examples:

```yaml
review_agent:
  outcome: review_completed
  quality: [precision, recall, f1, acceptance_rate]

fix_agent:
  outcome: merged_pr
  quality: [revert_rate, escaped_defects]

incident_agent:
  outcome: alert_triaged
  quality: [mttr, false_escalation_rate]
```

Report `cost / outcome`, not only `cost / token`.

### 2. Build benchmarks from the agent's real workload

Keep a representative set of historical tasks with known outcomes. For review, include real PRs containing known bugs and score candidate model/harness configurations on precision, recall, F1, latency, timeout rate, noise and cost.

Select the Pareto frontier rather than one globally "best" model.

### 3. Route narrow subagents to cheaper models by default

Use the strongest model where decomposition or difficult judgement is needed, but default bounded subagents to a cheaper model when benchmarks show they preserve quality.

```text
primary model -> decomposition / final judgement
cheap model   -> bounded search / extraction / repetitive checks
```

Allow overrides for tasks that benchmark poorly on the cheaper tier.

### 4. Treat context and tool plumbing as unit-economics problems

Track at least:

- input/output tokens per request;
- prompt-cache hit rate;
- preloaded instruction/tool-schema tokens;
- requests per turn;
- context size over time;
- cost per active session hour.

Large static tool catalogs are a recurring tax. Prefer dynamic tool search or CLI/gateway resolution when the model only needs a small subset of a large catalog.

### 5. Move chatty deterministic loops out of the model loop

If a workflow requires polling, pagination, filtering or joining deterministic results, execute that loop in code and return only the useful result to the model.

```text
bad:  model -> poll -> model -> poll -> model -> fetch -> model
better: model -> script(poll/filter/fetch) -> compact result -> model
```

Uber measured more than 50% token reduction even on small SQL examples, and much larger savings for bulk or very wide results. Treat the exact percentages as workload-specific, but the architectural lever is reusable.

### 6. Compare model changes with workload held constant

When evaluating a model/harness change, run the same benchmark corpus before and after. Otherwise adoption, workload mix and model capability changes can make fleet-wide spend trends misleading.

### 7. Surface cost where decisions happen

Expose live session cost and post-run diagnostics to developers and platform owners. Flag concrete anti-patterns such as:

- frontier model used for simple repetitive work;
- huge tool responses persisting in context;
- prompt-cache expiry after idle gaps;
- oversized startup instructions/tool schemas.

Prefer guidance and tiered budgets before blunt hard caps unless the workload requires strict limits.

## Why it is useful

The core decision rule is:

> Optimize **cost per accepted outcome at a required quality level**, not tokens in isolation.

That makes model routing, context engineering and tool design comparable engineering decisions instead of vendor-pricing folklore.

## Failure modes and caveats

- A cheaper model can increase retries, review noise or human repair cost; benchmark the full outcome.
- Acceptance rate is not proof of correctness. Pair behavioral telemetry with known-answer benchmarks where possible.
- Prompt-cache economics are provider-specific and change over time.
- A global benchmark can hide language/repository/task segments where one model performs badly.
- Do not optimize away useful context merely to reduce tokens; measure correctness after every change.

## Prototype experiment

For one existing coding or review agent:

1. collect 50-100 representative historical tasks;
2. define one outcome metric and 2-3 quality metrics;
3. run two model configurations plus the current baseline;
4. measure cost/outcome, quality, latency, retries and token drivers;
5. repeat after replacing static tool injection with search/on-demand loading or moving a polling loop into code;
6. adopt only configurations on the quality/cost Pareto frontier.

## Sources

- https://www.uber.com/pr/en/blog/efficient-software-factory/

## Related

- [Search-based code optimization agents with hard fitness gates](search-based-code-optimization-agents.md)
- [OpenAI Programmatic Tool Calling](openai-programmatic-tool-calling.md)
- [Spring AI ToolSearchToolCallingAdvisor](../spring/spring-ai-tool-search-advisor.md)
