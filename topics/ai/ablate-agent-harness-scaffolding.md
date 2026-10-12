# Ablate coding-agent harness scaffolding when models change

## What it is

Treat every non-safety harness component as a hypothesis about a model limitation, not permanent architecture. When the underlying model changes materially, re-run representative tasks and remove or simplify one scaffold at a time if it no longer improves outcomes enough to justify its cost and complexity.

Anthropic reported exactly this effect in its long-running application harness: context resets were important for Sonnet 4.5, became unnecessary with stronger models, and later sprint decomposition and evaluator passes became task-dependent rather than universally useful.

## Use when

Use this when:

- upgrading the primary coding model;
- changing context-window or compaction behavior;
- a harness has accumulated planners, reviewers, retries, resets or decomposition layers;
- latency/token cost is growing faster than task quality;
- a model release claims better planning, long-context retrieval, debugging or self-review.

Do **not** apply this rule to security or authorization boundaries such as sandboxing, credential scoping, exact side-effect approval, audit logging or branch protection. Those exist because the system is untrusted, not because a particular model is weak.

## The rule

```text
model upgrade
  -> stable task/eval set
  -> baseline full harness
  -> remove ONE scaffold
  -> repeat enough runs for stochastic variance
  -> compare quality + failures + cost + latency
       -> material regression: restore it
       -> no material regression: keep the simpler harness
  -> test the next scaffold
```

A scaffold can include:

- context reset / explicit handoff;
- planner agent;
- task or sprint decomposition;
- independent evaluator;
- self-review pass;
- retry loop;
- extra retrieval stage;
- subagent fan-out.

## Example ablation table

Keep the benchmark tied to real work rather than generic coding scores.

| Variant | Task success | Escaped defects | Human repair min | p95 latency | Cost/task |
|---|---:|---:|---:|---:|---:|
| full harness | ... | ... | ... | ... | ... |
| no context reset | ... | ... | ... | ... | ... |
| no planner | ... | ... | ... | ... | ... |
| no evaluator | ... | ... | ... | ... | ... |

Run each stochastic variant multiple times or across enough tasks that a lucky single trajectory cannot decide the architecture.

## Why one component at a time

Removing several layers simultaneously makes it hard to identify what was load-bearing. Anthropic explicitly moved from broad simplification attempts to methodical one-component-at-a-time removal because otherwise the source of regressions was unclear.

This also makes harness evolution reversible: each layer has evidence for why it still exists.

## Capability-triggered scaffolding

Some layers should become conditional rather than globally enabled.

For example:

```text
simple / well-covered task
  -> generator + deterministic tests

edge-of-capability task
  -> planner
  -> generator
  -> independent evaluator
  -> repair loop
```

Anthropic found an evaluator remained useful for work near the model's capability boundary while becoming unnecessary overhead for easier work after the model improved. The reusable decision is to route by measured task difficulty/reliability, not by habit.

## Model × task-family calibration

A September 2026 controlled study compared two harnesses across ten models and three task families (issue repair, repository generation and repository-centric tasks), then isolated structured tools, compaction, planning, general/task-specific subagents and lazy skills. The authors found harness benefits were neither uniform nor monotonic: some richer harness features became less helpful for stronger models on localized repair, while structured tool use and task-specific subagents were more dependable on their repository-generation set. General subagents and compaction could hurt that task family.

**Do not extrapolate an ablation win from one task class to another.** Freeze a matrix of model version × task type × single changed component, for example:

| Model | Task family | Harness variant | Success | Escaped defects | Human repair | p95 | Tokens |
|---|---|---|---:|---:|---:|---:|---:|
| M1/M2 | localized bug fix | baseline / no compaction | ... | ... | ... | ... | ... |
| M1/M2 | multi-file migration | baseline / specialist subagent | ... | ... | ... | ... | ... |
| M1/M2 | repository generation | baseline / generic subagent | ... | ... | ... | ... | ... |

Keep security and authorization constraints constant. Prefer task-specific subagents with bounded contexts/tools when the task justifies them, but validate on our own tasks before changing defaults. The paper's scores are from a bounded model/benchmark sample, not a universal ranking of features.

Source: https://arxiv.org/abs/2609.32459

## Failure modes / caveats

- **Benchmark drift:** an ablation set that contains only easy tasks will incorrectly tell you that verification layers are useless.
- **Safety conflation:** never remove permission, sandbox or approval controls because a model looked reliable in an eval.
- **One-run conclusions:** agent behavior is stochastic; repeat runs or use a sufficiently broad task set.
- **Model-specific folklore:** a workaround needed by one model/version can become cargo cult after an upgrade.
- **Hidden quality dimensions:** include human review burden, escaped defects and scope adherence, not just tests passing.

## Prototype experiment

On the next model upgrade, take 20-50 recent agent tasks with known outcomes and run:

1. current production harness;
2. current harness minus one expensive scaffold;
3. repeat for each candidate scaffold only after the prior comparison is resolved.

Record task success, defect escape, human correction time, cost and latency. Keep a short `why-this-layer-exists.md` or eval record beside each retained harness component.

## Sources

- https://www.anthropic.com/engineering/harness-design-long-running-apps
- https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents

## Related

- [Deterministic outer loop for coding-agent platforms](deterministic-outer-loop-agent-platform.md)
- [Precision-first multi-stage AI code review](precision-first-multi-stage-code-review.md)
