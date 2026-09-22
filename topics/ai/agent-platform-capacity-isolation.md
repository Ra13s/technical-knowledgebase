# Protect shared agent platforms with workload classes and bounded recovery

## What it is

A reliability pattern for agent platforms at scale: model the entire agent trajectory as shared-infrastructure demand, attach stable workload identity to every step, allocate capacity by workload class, and bound retries both per trajectory and across the shared service.

Datadog describes this from production experience running agentic SDLC and operational workflows. The important shift is that a well-behaved individual agent can still create a badly behaved platform when many agents compete for model quota, queues, CI workers, sandboxes and downstream APIs at machine speed.

## Use when

Use this when multiple teams or agents share any constrained dependency, especially:

- LLM/provider quotas;
- tool gateways or MCP services;
- background queues/workers;
- CI executors;
- browser/code sandboxes;
- downstream APIs;
- human-approval queues.

This becomes necessary before adoption is large enough that retries and parallelism can turn ordinary throttling into an outage.

## Architecture

```text
agent run / task
   |
   +--> stable identity: workload, owner, env, service-class, run-id
   |
   v
trajectory inventory
   -> model calls
   -> tool calls
   -> queues
   -> CI jobs
   -> sandboxes
   -> downstream APIs
   -> side effects
        |
        v
per-dependency capacity signals
        |
        v
workload-class policy
   -> reserve / continue
   -> queue
   -> reduce concurrency
   -> fallback
   -> reject
        |
        v
bounded trajectory + service recovery
```

## Implementation rules

### 1. Inventory the full trajectory

Do not capacity-plan only the model provider. Record every dependency an agent can consume from trigger to final side effect.

For one representative workflow, keep something like:

```yaml
trajectory:
  - dependency: llm-provider
    owner: ai-platform
    constraint: tokens_per_minute
  - dependency: sandbox-pool
    owner: dev-platform
    constraint: active_slots
  - dependency: ci
    owner: build-platform
    constraint: workers
  - dependency: github-api
    owner: external
    constraint: requests_per_hour
```

A conversational agent and a long-running coding agent may touch the same services but have completely different pressure profiles.

### 2. Measure each dependency by its actual limiting resource

Useful signals include:

- requests/minute and tokens/minute;
- in-flight concurrency and duration;
- queue depth, oldest item age and drain time;
- tool-call/retry amplification factor;
- CI worker and sandbox utilization;
- downstream/provider quota headroom;
- remaining per-run token/cost budget;
- latency signals such as time-to-first-token when they indicate pressure.

Do not rely only on fleet-wide aggregates. Break signals down by workload and dependency so a large evaluation run cannot hide inside a healthy-looking total.

### 3. Put a stable workload identity on every request

At minimum propagate:

```text
workload
owner
environment
service_class
task_or_run_id
```

Carry these through queues, tool calls, traces, logs, audit events and downstream actions. Preserve identity on rejected/429 requests too; losing identity at the enforcement boundary makes incidents hardest to diagnose precisely where the system is under stress.

Prefer non-human or delegated identities with short-lived/workload-bound credentials over shared developer keys.

### 4. Define workload classes before contention

For every shared constrained dependency, decide what each class does under pressure.

```yaml
classes:
  customer-facing:
    reserved_capacity: true
    overload: queue-short-then-reject
  production-background:
    reserved_capacity: partial
    overload: reduce-concurrency
  evaluation:
    reserved_capacity: false
    overload: pause
```

Possible actions are **continue, queue, reduce concurrency, fallback or reject**. Make the choice explicit instead of letting the loudest caller win.

When isolation matters enough, separate provider accounts/buckets or worker pools so evaluation traffic cannot consume production headroom.

### 5. Bound recovery twice

A per-agent retry budget is necessary but insufficient.

**Trajectory level:** cap attempts, wall-clock time and/or cost. Once exhausted, enter a terminal state rather than looping indefinitely.

```yaml
run_recovery:
  max_attempts: 3
  max_duration: 45m
  max_cost_usd: 8
```

**Service level:** bound aggregate recovery with concurrency limits, queues, circuit breakers, load shedding or overload signals.

Ten agents each allowed three retries can still become thirty executions before tool fan-out is counted.

### 6. Classify failures as retryable or terminal

Respect `429`/`Retry-After`, use exponential backoff with jitter for transient failures, and label policy/authorization violations as non-retryable so a guardrail rejection cannot trigger an automated retry storm.

Write a failure/recovery contract for each major dependency:

```yaml
failure_contract:
  retryable: [429, 502, 503]
  terminal: [policy_denied, invalid_request]
  trajectory_budget: 3
  service_concurrency_limit: 100
  overload_action: queue
```

### 7. Use lifecycle-specific timeouts

Separate timeouts for connection setup, model streaming, tool activity, workflow steps and whole runs. Long-running requests also need deployment/shutdown behavior that lets in-flight work complete or checkpoint safely.

A five-second HTTP-service shutdown default may be fine for ordinary APIs and disastrous for streamed model requests. Measure real p99 lifetimes rather than inheriting generic defaults blindly.

### 8. Correlate intent to side effects

Use the same run/workload identity for observability and enforcement. Record:

- initiating task/context;
- action and target resource;
- authorization/approval used;
- final outcome;
- downstream audit correlation IDs.

This gives incident responders a trace from "why did the agent start?" to "what changed?" without storing private chain-of-thought.

## Why it is useful

Agent reliability becomes an aggregate-systems problem before individual agents look broken. Capacity classes and dual recovery budgets let the platform protect critical work while allowing cheaper/background work to degrade gracefully.

The reusable rule is:

> **Every autonomous run needs both a bounded personal budget and a platform policy for what happens when everyone retries at once.**

## Evidence

Datadog reports using this approach while operating agentic SDLC/operations workloads. In one projected internal coding-agent workload, roughly 2,000 requests/minute appeared acceptable, but projected token consumption (~45 billion tokens/day) would have approached an upstream throughput limit; dependency-specific capacity modeling caught that before scale-out.

Datadog also separates evaluation and production provider accounts and uses finer-grained rate-limit buckets within production. Treat the exact topology and numbers as organization-specific evidence, not universal sizing guidance.

## Caveats

- Reserved capacity can strand resources; measure utilization and revisit classes.
- A fallback model/tool can change correctness semantics, not only latency/cost; benchmark fallback behavior.
- Queuing can turn an outage into hours of stale work. Track oldest-item age and define expiry.
- Retry budgets must account for fan-out: one model retry may trigger many tool/API/CI operations.
- Stable workload IDs are operational metadata, not an excuse to log sensitive prompts or hidden reasoning.

## Prototype experiment

Pick one background coding agent and:

1. inventory every dependency from trigger to PR/side effect;
2. attach `workload`, `owner`, `environment`, `service_class` and `run_id` to every hop;
3. define one limiting metric + warning/critical threshold for each dependency;
4. create at least two workload classes (production vs evaluation/background);
5. add per-run attempt/time/cost limits and one service-level concurrency/queue limit;
6. simulate a provider 429 storm and sandbox/CI exhaustion;
7. verify low-priority work sheds load while critical work retains headroom and rejected requests remain attributable.

## Sources

- Datadog Engineering, *How to operate shared platforms safely at agent scale* (2026-09-15): https://www.datadoghq.com/blog/operating-shared-platforms-agent-scale/

## Related

- [Engineer agent fleets with outcome-denominated unit economics](agent-fleet-unit-economics.md)
- [Deterministic outer loop for coding-agent platforms](deterministic-outer-loop-agent-platform.md)
- [Bind agent approvals to the exact side effect](enforcement-bound-agent-approvals.md)
- [Contain coding agents at the environment boundary first](environment-first-agent-containment.md)
