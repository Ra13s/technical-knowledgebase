# Search-based code optimization agents with hard fitness gates

## What it is

For engineering tasks with an objective fitness function, use the LLM as a **candidate generator inside a search loop**, not as a one-shot optimizer. A deterministic harness compiles/runs each candidate, rejects incorrect variants, measures fitness, returns diagnostics, and lets the search policy decide what to explore next.

Meta's production KernelEvolve system uses this shape for hardware kernels: graph/tree search over generated implementations, hard correctness checks, performance profiling, runtime-driven retrieval, persistent optimization knowledge, and explicit termination rules.

## Use when

Use this pattern when all of the following are true:

- many valid implementations can satisfy the same contract;
- there is a reliable executable correctness oracle;
- quality is measurable with an objective function;
- trying many candidates is cheaper than expert manual search;
- diagnostics can explain why a candidate failed or underperformed.

Examples beyond accelerator kernels can include:

- SQL/query rewrites with result-equivalence tests + latency/cost fitness;
- compiler or serialization hot-path optimization;
- algorithm/configuration tuning with a deterministic benchmark;
- memory/layout tuning;
- build or scheduling configuration where outcomes can be replayed safely.

Do not use it for requirements, UX, architecture choices or other tasks where the fitness function is mostly subjective or incomplete.

## Loop

```text
spec + reference behavior + search budget
  -> retrieve constraints / prior successful patterns
  -> generate N candidates
  -> compile / execute
  -> correctness gate
       -> fail: keep diagnostics, fitness = reject
       -> pass: benchmark/profile
  -> rank/select promising candidates
  -> mutate/refine/restart
  -> repeat until target, budget, or stagnation stop
  -> return best fully validated candidate
```

Minimal pseudocode:

```python
frontier = seed_candidates(spec)
best = None

while budget.remaining() and not target_met(best):
    evaluated = []
    for candidate in frontier:
        result = validate(candidate, reference_cases)
        if not result.correct:
            evaluated.append(reject(candidate, result.diagnostics))
            continue

        perf = benchmark(candidate, representative_workload)
        evaluated.append(score(candidate, perf, result.diagnostics))

    best = select_best(best, evaluated)
    context = diagnostics_and_retrieval(evaluated, knowledge_base)
    frontier = search_policy.expand(evaluated, context)

return best
```

## Hard rule: correctness before optimization

Never let a faster wrong candidate win.

```text
fitness(candidate) = INVALID
unless reference-equivalence checks pass
```

The correctness oracle should cover the real contract, not only a convenient happy-path sample. Meta validates generated kernels against reference implementations before considering performance.

## Feed structured diagnostics, not only a scalar score

A score tells the agent which candidate won; diagnostics tell it what to try next.

Useful signals can include:

- compiler errors;
- failing reference cases;
- profiler bottlenecks;
- allocation counts;
- query plans;
- cache/memory pressure;
- representative workload slices where performance regressed.

KernelEvolve feeds hardware utilization and bottleneck diagnostics back into prompt synthesis instead of reducing evaluation to “candidate A is 1.2x faster.”

## Retrieve knowledge from runtime signals

Do not inject every optimization document into every attempt. Retrieve the relevant constraints when evidence points to them.

Example:

```text
compiler error      -> language/compiler debugging knowledge
memory bottleneck   -> memory-layout guidance
specific platform   -> platform-specific constraints
stagnating branch   -> sibling/alternative strategy history
```

This is useful for proprietary or newly introduced systems that the base model could not have learned during training.

## Preserve successful search knowledge

When an optimization repeatedly works, distill the technique into a reusable skill/knowledge entry rather than requiring every future search to rediscover it.

Keep provenance: target, workload, platform/version, evidence, and cases where the technique failed. A “successful trick” without its applicability boundary becomes future prompt folklore.

## Termination rules

Autonomous search must stop deterministically. Typical gates:

```yaml
stop_when:
  target_speedup: 1.20
  max_candidates: 200
  max_cost_usd: 50
  wall_clock_minutes: 90
  no_improvement_rounds: 4
```

The outer harness, not the LLM, owns the budget.

## Why it is useful

LLMs are good at proposing qualitatively different implementations; deterministic systems are good at proving correctness and measuring objective outcomes. Search lets each do the job it is suited for.

The pattern also supports deliberate exploration: refine a successful parent, compare siblings, or restart from a clean branch when the search gets trapped in a local optimum.

## Failure modes / caveats

- A bad benchmark produces a very efficiently optimized mistake.
- Microbenchmark wins may regress representative production distributions; include real input shapes/workloads.
- Search can be computationally expensive; cap candidates, cost and wall time.
- Correctness tests are only as strong as the reference oracle.
- Persisted optimization knowledge can become stale across platform/compiler changes; version it.
- Meta's published performance numbers demonstrate feasibility for kernel optimization, not expected gains for unrelated codebases.

## Prototype experiment

Choose one bounded performance problem with a reference test suite, for example a slow SQL transformation or serialization hotspot. Compare:

1. one-shot “optimize this” agent;
2. five independent samples;
3. bounded search with correctness gate + profiler diagnostics + iterative refinement.

Measure best valid performance, invalid-candidate rate, agent cost, wall time and human tuning time.

## Sources

- https://engineering.fb.com/2026/04/02/developer-tools/kernelevolve-how-metas-ranking-engineer-agent-optimizes-ai-infrastructure/
- https://arxiv.org/abs/2512.23236

## Related

- [Deterministic outer loop for coding-agent platforms](deterministic-outer-loop-agent-platform.md)
- [Evidence-gated coding-agent edits](evidence-gated-coding-agent-edits.md)
