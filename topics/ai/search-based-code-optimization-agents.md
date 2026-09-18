# Search-based code optimization agents with hard fitness gates

## What it is

For engineering tasks with an objective fitness function, use the LLM as a **candidate generator inside a search loop**, not as a one-shot optimizer. A deterministic harness compiles/runs each candidate, rejects incorrect variants, measures fitness, returns diagnostics, and lets the search policy decide what to explore next.

Meta's production KernelEvolve system uses this shape for hardware kernels: graph/tree search over generated implementations, hard correctness checks, performance profiling, runtime-driven retrieval, persistent optimization knowledge, and explicit termination rules.

Datadog's DODO adds an important prerequisite for ordinary production code: before optimizing against a benchmark, first use production inputs and profiler shape to prove that the benchmark itself behaves like production. Then freeze that benchmark so the optimizer cannot game its fitness function.

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

For a benchmark that is already trustworthy:

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

## Ground the fitness function before you optimize

A deterministic benchmark can still be the wrong oracle. If its input distribution or execution shape differs from production, the agent will optimize the benchmark successfully and the service unsuccessfully.

For production hotspots, split the system into **two loops**:

```text
production inputs + profiler
        |
        v
benchmark-generation loop
  generate benchmark
  -> profile it
  -> compare execution shape with production
  -> adjust inputs/setup
  -> stop at similarity target / budget
        |
        v
freeze trusted benchmark
        |
        v
code-optimization loop
  edit service code only
  -> correctness tests
  -> frozen benchmark
  -> keep best valid patch
```

Datadog's DODO uses real sampled invocations plus a production CPU call tree, iterating the benchmark until its profile reaches at least 98% similarity to production before optimization begins. Treat 98% as their operating threshold, not a universal constant.

### Capture state as well as arguments

For hot paths, realistic behavior can depend on receiver/internal state such as caches, rules, lookup tables or configuration. Reconstructing only the method arguments may still produce a synthetic execution shape.

When safe and privacy-appropriate, capture representative invocation state or reproduce it deterministically from production-derived fixtures.

### Match the hardware when the fitness depends on it

CPU profile similarity and microbenchmark timing can vary by architecture. DODO filters production profiles by CPU architecture and runs benchmarks on matching hardware.

If the target metric is hardware-sensitive, include hardware/runtime identity in the benchmark contract rather than assuming results transfer between arm64 and amd64 or different JDK/compiler configurations.

### Freeze the benchmark before candidate search

The benchmark-generation agent and optimization agent should have different write boundaries:

```text
benchmark agent  -> may write benchmark/fixture, not service code
optimizer        -> may write service code, benchmark is read-only
```

Otherwise the easiest way to improve the score may be to weaken or rewrite the fitness function.

### Fingerprint flakes before optimization

Run the correctness suite repeatedly before the search to identify pre-existing flakes. A candidate should not be rewarded or rejected because the baseline itself is nondeterministic.

Datadog runs tests three times before the optimization loop and surfaces known flakes to the agent.

## Feed structured diagnostics, not only a scalar score

A score tells the agent which candidate won; diagnostics tell it what to try next.

Useful signals can include:

- compiler errors;
- failing reference cases;
- profiler bottlenecks;
- allocation counts;
- query plans;
- cache/memory pressure;
- representative workload slices where performance regressed;
- production-vs-benchmark call-path divergences.

KernelEvolve feeds hardware utilization and bottleneck diagnostics back into prompt synthesis instead of reducing evaluation to “candidate A is 1.2x faster.” DODO similarly returns profile divergences such as missing, over-exercised and under-exercised call paths while constructing its benchmark.

## Keep the best validated state, not merely the last state

Iterative optimizers often regress after finding a better candidate. Snapshot every validated candidate and retain the best observed state according to the frozen fitness function.

```text
candidate 17 = 1.18x
candidate 18 = 1.31x  <- best
candidate 19 = 1.22x
final output = candidate 18
```

DODO stores numbered patches and retains the lowest observed benchmark time so a later edit cannot erase an earlier gain.

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

Production-grounded fitness adds a second safety rule: **prove the benchmark resembles the workload before trusting the optimizer**. Datadog reports that three deployed DODO optimizations reduced more than 8% of one mature service's total CPU cost; one successful optimization depended directly on a production input distribution that a synthetic benchmark could easily have missed. Those results demonstrate the value of grounding for that service, not an expected speedup elsewhere.

The pattern also supports deliberate exploration: refine a successful parent, compare siblings, or restart from a clean branch when the search gets trapped in a local optimum.

## Failure modes / caveats

- A bad benchmark produces a very efficiently optimized mistake.
- Microbenchmark wins may regress representative production distributions; include real input shapes/workloads.
- Production-derived samples can contain sensitive data; sanitize/minimize them before turning them into fixtures or prompts.
- Profile similarity is not correctness. Keep functional tests as a separate hard gate.
- Search can be computationally expensive; cap candidates, cost and wall time.
- Correctness tests are only as strong as the reference oracle.
- Persisted optimization knowledge can become stale across platform/compiler changes; version it.
- Meta's and Datadog's published performance numbers demonstrate feasibility in their workloads, not expected gains for unrelated codebases.

## Prototype experiment

Choose one bounded performance problem with a reference test suite, for example a slow SQL transformation or serialization hotspot.

First compare benchmark quality:

1. write the current synthetic/local benchmark;
2. collect safe representative production inputs plus profiler/trace shape;
3. measure how closely the local benchmark reproduces production branches/cost distribution;
4. iterate only the benchmark until an explicit similarity target is met, then freeze it.

Then compare optimization strategies:

1. one-shot “optimize this” agent;
2. five independent samples;
3. bounded search with correctness gate + frozen production-grounded benchmark + profiler diagnostics + iterative refinement.

Measure best valid performance, invalid-candidate rate, benchmark-to-production prediction error, agent cost, wall time and human tuning time.

## Sources

- Meta Engineering, *KernelEvolve: How Meta's ranking engineer agent optimizes AI infrastructure* (2026-04-02): https://engineering.fb.com/2026/04/02/developer-tools/kernelevolve-how-metas-ranking-engineer-agent-optimizes-ai-infrastructure/
- KernelEvolve paper: https://arxiv.org/abs/2512.23236
- Datadog Engineering, *Why AI code optimization needs production-grounded benchmarks* (2026-06-08): https://www.datadoghq.com/blog/ai/production-grounded-code-optimization/

## Related

- [Deterministic outer loop for coding-agent platforms](deterministic-outer-loop-agent-platform.md)
- [Evidence-gated coding-agent edits](evidence-gated-coding-agent-edits.md)
