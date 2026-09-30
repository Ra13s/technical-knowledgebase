# Profile agent fleets by semantic operation

## What it is

A profiling pattern for long-running agent systems: aggregate many trajectories by **semantic operation** such as planning, code search, test repair or review instead of looking only at one trace at a time.

Traditional tracing answers “what happened in this run?”. Semantic profiling answers fleet questions such as:

- which task families consume the most tokens or wall time;
- which repeated operations touch the most files or network resources;
- where failures cluster across many runs;
- which workflow layer should be optimized next.

AgentPProf demonstrates this by replacing a CPU call stack with an operation stack and emitting pprof-compatible profiles.

## Use when

Use this when you already have agent traces/session logs but still cannot answer where cost, latency or repeated work accumulates across a fleet.

It is especially useful for:

- long-running coding agents;
- recurring CI/review/remediation agents;
- multi-agent workflows where subagents may duplicate exploration;
- comparing harness changes across a stable workload;
- finding repeated tool sequences worth turning into a deterministic primitive or playbook.

Do not use semantic profiling as a substitute for authoritative executor telemetry. A semantic profile is an aggregation/projection over events; it does not prove that every filesystem, network or production side effect was captured.

## Data model

Normalize raw trajectory events into a small semantic stack:

```text
task
  -> skill/workflow
     -> phase
        -> action
           -> object
              -> outcome
```

Attach one or more weights to each operation:

```yaml
weights:
  - input_tokens
  - output_tokens
  - elapsed_ms
  - operation_count
  - file_effects
  - network_effects
```

Then aggregate identical semantic stacks across many runs. Repeated behavior becomes wider in the profile instead of remaining thousands of isolated spans.

## Concrete workflow with AgentPProf

Install:

```bash
cargo install agentpprof
```

Profile a repository's recorded Codex/Claude Code sessions by token use:

```bash
agentpprof --project-root . --view tokens -o tokens.pb.gz
go tool pprof -top tokens.pb.gz
go tool pprof -http=:0 tokens.pb.gz
```

Or inspect wall time:

```bash
agentpprof --project-root . --view time -o time.svg
```

For a reproducible comparison, freeze the source sessions rather than letting the tool discover an ever-changing recent-history set:

```bash
agentpprof \
  --project-root /work/repo \
  --session-file ~/.codex/sessions/.../session.jsonl \
  --session-file ~/.claude/projects/.../session.jsonl \
  --view tokens \
  -o tokens.svg
```

Record the profiler version, project revision, session identifiers, tagging rules, stack definition and selected weight alongside the result.

## Implementation rules

### 1. Separate chronology from aggregation

Keep ordinary traces for sequence-level debugging. Build a second profiling view for aggregation.

```text
raw trace -> debug one run
         \
          -> normalize/tag -> aggregate -> find fleet hotspot
```

Do not force one visualization to do both jobs.

### 2. Prefer stable deterministic tags before semantic inference

If a workflow already has stable IDs such as `review`, `test-repair`, `tool-search` or `publish`, map them directly.

Use a learned/LLM tagger only for the residual free-form activity that cannot be classified reliably from structured events. Keep the raw event reference so a suspicious aggregate can be inspected.

### 3. Freeze the corpus before comparing harness versions

A before/after flamegraph is meaningless if the workload mix changed.

Use the same task corpus or explicit session set when evaluating:

- a new model;
- dynamic tool loading;
- compaction;
- subagent routing;
- retrieval changes;
- prompt/cache layout.

### 4. Profile several weights, not only tokens

A token hotspot can be cheap cached input; a small-token operation can dominate wall time or downstream effects.

At minimum inspect:

```text
tokens
wall time
operation count
file effects
network/tool effects
failure count
```

### 5. Turn wide stacks into experiments, not immediate deletions

For each hotspot, form a concrete hypothesis:

```text
wide code-search stack
  -> hypothesis: repeated exploration
  -> experiment: semantic retrieval / cached repository map

wide tool-schema/setup stack
  -> hypothesis: static context tax
  -> experiment: progressive tool disclosure

wide retry/test-repair stack
  -> hypothesis: flaky oracle or weak failure evidence
  -> experiment: deterministic failure classifier + bounded retry
```

Re-run the frozen corpus and keep the change only if outcome quality holds.

### 6. Keep profiling and audit evidence separate

Semantic labels are useful derived data, not authoritative evidence.

Maintain links from the profile back to trusted model/tool/executor events, especially when analyzing security-sensitive or production-impacting behavior.

## Why it is useful

Agent traces become hard to reason about once there are hundreds or thousands of long sessions. Profiling gives the same kind of “where is the system spending itself?” view that CPU profilers provide for programs, but attributes cost and behavior to task intent rather than code frames.

The reusable rule is:

> **Trace to explain one run; profile to decide what to optimize across runs.**

## Evidence and caveats

The September 2026 AgentPProf paper reports:

- `0.764` B³ F1 against human operation-segmentation annotations on CodeTraceBench;
- up to 56% improvement in mean average precision on three problem-localization benchmarks when using the generated profiles.

These are research benchmark results, not evidence that automatic semantic segmentation is correct for our repositories or agents.

Other caveats:

- tag drift can make historical comparisons misleading;
- a profile cannot attribute events that were never recorded;
- aggregating across unrelated workload classes can hide the real hotspot;
- LLM-based tagging can add cost and nondeterminism;
- pprof width is only as meaningful as the chosen weight and input corpus.

## Prototype experiment

Take 50-100 completed coding-agent sessions from one repository.

1. Freeze the exact input sessions and repository revision.
2. Generate token and time profiles.
3. Inspect the top five semantic stacks.
4. Pick one hotspot with a plausible harness fix.
5. Apply one change only.
6. Replay or rerun the same workload.
7. Compare task success, cost, latency, retries and human correction as well as profile width.

Promote the change only if the outcome metric stays at least as good.

## Sources

- Zheng et al., *AgentPProf: Semantic Profiler for Long Horizon AI Agents* (2026-09-14): https://arxiv.org/abs/2609.20301
- AgentSight, *Build an Agent Flamegraph*: https://agentsight.us/guides/agent-flamegraph/
- AgentPProf source: https://github.com/eunomia-bpf/agentsight

## Related

- [Externalize authoritative agent telemetry](external-authoritative-agent-telemetry.md)
- [Engineer agent fleets with outcome-denominated unit economics](agent-fleet-unit-economics.md)
- [Deterministic outer loop for coding-agent platforms](deterministic-outer-loop-agent-platform.md)
