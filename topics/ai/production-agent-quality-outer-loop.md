# Diagnose silent production agent regressions with an evidence-gated quality outer loop

## What it is

Run an **out-of-request-path** quality investigator over sampled production agent trajectories. Detect semantic failures that still return HTTP 200 (skipped tool checks, dropped constraints, broken handoffs), group recurring issues, independently verify each candidate against original transcripts, and diagnose using the **immutable code/prompt/tool snapshot actually deployed**.

Google's open-source AQuA (Ambient Quality Agent, published October 8, 2026) is an executable reference. It is a **quality investigation and human-triage loop**, not an autonomous production patcher or replacement for deterministic checks.

## Use when

- an agent returns superficially successful responses while failing user goals;
- failures span several subagents, tools or turns and ordinary traces lack an issue summary;
- prompt/skill/tool versions change often and offline evals miss live workload drift;
- high-volume telemetry makes manual transcript reading prohibitively expensive.

Do **not** silently export raw customer transcripts to a third-party model. Pseudonymize, minimize, restrict access, and obey retention/privacy requirements before enabling an investigator.

## Five-stage pipeline

~~~text
scheduled / post-deploy trigger
  -> SAMPLE: bounded stratified production sessions (up to 1000/run in AQuA)
  -> REVIEW: read real tool/state trajectory against goal.md + deterministic checks
       -> structured {actual, expected, trace refs, reviewer config}
  -> CLUSTER: group the same mechanism, not merely similar symptoms
  -> VERIFY: independent judge inspects up to 3 original full transcripts/cluster
       -> reject, defer or accept, with linked evidence
  -> TRACK: NEW / RECURRING / RESOLVED + owner, occurrences, last-seen
  -> ON-DEMAND ROOT CAUSE: trace + immutable deployed source snapshot
       -> validated file:line evidence or external/tool/handoff attribution
  -> human triage / regression replay / code branch / CI / separate merge gate
~~~

### A concrete minimal implementation

1. Instrument session and tool events with stable IDs: `session_id`, span parent ID, `deployment_revision`, `agent_id`, `tool_name`, tool inputs/result summaries, state/handoff changes and error codes. Keep evidence in an access-controlled, append-only telemetry store; store only data needed to evaluate quality.
2. At deployment, capture a **read-only hash-addressed snapshot** of prompts, tool schemas, skills, orchestrator configuration and source files. Do not let a current branch or PR supply the diagnosis snapshot.
3. Write `goal.md` describing actual product invariants, e.g. “confirm booking only after availability tool success” or “copy user restrictions across delegation.” Explicitly exclude subjective style nits from issue generation.
4. Sample with random coverage **and** a bounded anomaly-focused slice. Keep denominators and sampling rules: an anomaly-only sampler cannot estimate production prevalence.
5. Run a low-cost reviewer to produce schema-valid `actual/expected` findings with references to spans and required tool/state evidence. Group issues by mechanism; require a second reviewer/verifier to inspect the full original trajectories before creating a durable issue.
6. Reuse a stable issue key for recurring clusters, but do not infer complete resolution merely because the issue was absent from a small sample. Maintain an explicit `not_observed` status and replay a held-out regression set before closing critical cases.
7. Diagnose only on demand or for high-priority issues. Validate each proposed `path:start-end` against the pinned deployment snapshot; any invalid citation fails the diagnosis. A suspected external dependency failure should cite the failing handoff/tool evidence instead of inventing a source edit.
8. Export a scrubbed, reproducible test trajectory to CI. Model-proposed patches must still traverse normal edit permissions, security review, branch protections, CI and human gates.

Example finding record:

~~~json
{
  "deployment_revision": "<sha>",
  "issue_id": "missing-seat-availability-check",
  "status": "NEW",
  "rubric": "verify_seat_before_confirming",
  "actual": "seat saved without selection/availability tool response",
  "expected": "selection tool success before persisting seat",
  "occurrences": ["trace:session-a/span-b"],
  "verified_transcripts": 2,
  "source_citation": "planning/prompt.py:93-96",
  "source_citation_checked": true,
  "next_action": "replay-regression"
}
~~~

### Trying Google's reference implementation

The reference lives at `google/adk-recipes/core/python/ambient-quality-agent`. It expects Google Cloud telemetry (Cloud Trace/Logging or BigQuery), deployment snapshots in Cloud Storage, an isolated GCP project and Google ADK/agents-cli. The source's documented example flow is:

~~~sh
agents-cli extension add "PATH_TO_AQUA_CHECKOUT"
agents-cli infra single-project --project="YOUR_GOOGLE_CLOUD_PROJECT" --apply
agents-cli deploy --project="YOUR_GOOGLE_CLOUD_PROJECT" --region us-east1
# Optional: publish domain-specific deterministic metrics.
agents-cli aqua metrics publish
# Fetch an independently verified issue and relevant traces.
agents-cli aqua list-insights --status NEW --root-cause false
# After diagnosis:
agents-cli aqua get-insight <insight-id> > insight.json
~~~

These are **reference-project commands**, not a drop-in setup for a generic Spring app. Adapt the five-stage loop to an OTel+PostgreSQL/BigQuery backend if using another stack.

## Evidence and limits

In Google's 32-session *scripted replay* of four travel-assistant journeys, the first sweep found 42 session findings grouped into 9 issue candidates; independent verification rejected 3 clusters and retained 6. Two one-line prompt fixes reduced observed seat-check bypasses 15 -> 2 and lost vegan preferences 7 -> 0; total passing sessions went 5/32 -> 13/32. The untouched toolset contradiction persisted. This validates an example diagnosis loop, **not** production effectiveness or generalized precision/recall.

Google describes an additional 87-trace internal benchmark where the verifier rejected 4/24 candidate clusters. It samples at most 1,000 sessions and verifies only a few transcripts per cluster; rare or long-tail defects may be missed, and one cluster label does not prove all occurrences are true. Its automatic “RESOLVED after 14 days unseen” is a product heuristic, not proof of safety.

**Authority rule:** the investigator may classify and recommend; only the trusted build/publisher and humans (when required) may edit/merge. Evaluate the investigator on human-verified actionable findings per review minute, false positives, missed seeded regressions, mean detection delay and telemetry/model cost.

## Prototype experiment

Replay 200 scrubbed Spring AI agent sessions and inject five deliberate silent faults: skipped authorization/availability check, lost context in a subagent handoff, tool-schema mismatch, hallucinated confirmation, and stale tool result. Compare (A) generic LLM review of every session, (B) sampled reviewer+clustering, and (C) sampled reviewer+independent verifier+revision-pinned diagnosis. Include 20 counterexamples where user intent makes the apparently suspicious behavior correct. Record verified yield, missed critical classes, reviewer time, cost, source citation validity and false “resolved” rates.

## Sources

- Google Developers, *The Outer Loop, Insights First*, **2026-10-08**: https://developers.googleblog.com/the-outer-loop-insights-first-an-ambient-quality-agent-that-diagnoses-your-production-agent/
- Source code / recipe: https://github.com/google/adk-recipes/tree/main/core/python/ambient-quality-agent
- InfoQ's later analysis, **2026-11-13**: https://www.infoq.com/news/2026/11/google-ambient-agent-prod/

## Related

- [Externalize authoritative agent telemetry](external-authoritative-agent-telemetry.md)
- [Evidence-gated coding-agent edits](evidence-gated-coding-agent-edits.md)
- [Precision-first multi-stage AI code review](precision-first-multi-stage-code-review.md)
- [Spring AI session parts and replay](../spring/spring-ai-session-parts-replay.md)
