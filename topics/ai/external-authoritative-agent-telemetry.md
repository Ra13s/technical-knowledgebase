# Externalize authoritative agent telemetry

## What it is

An observability pattern for autonomous agents: keep the authoritative record of model calls, tool requests and actual executor outcomes in a control-plane component separate from the agent task workspace.

Local session files remain useful for UI and recovery, but they should not be the only evidence used for production debugging, evaluation or audit.

## Use when

Use this for unattended agents, production-impacting workflows, incident-sensitive automation, or any platform that needs to reconstruct what an agent actually did.

## Architecture

```text
model / tool gateway
       |
       v
trusted recorder ---> protected telemetry sink
       |
       v
agent runtime
```

## Implementation rules

- Record outside the task workspace using a separate service, host-side collector or provider-side audit stream.
- Keep recorder credentials outside the agent runtime.
- Carry stable run and operation identifiers across model calls, tool requests, policy decisions, executor invocations and executor results.
- Prefer executor/gateway evidence over an agent-authored summary for important tool outcomes.
- Preserve run/session identity, event ordering, trusted timestamps and model/tool identity.
- Apply deterministic redaction or tokenization before durable storage and version the redaction policy.
- Keep monitoring separate from authorization and sandbox enforcement.
- Periodically reconstruct representative runs from trusted recorder/executor evidence alone.

## Why it is useful

Observability is strongest when the component being observed does not own the only copy of the authoritative record.

## Caveats

- OpenTelemetry is a transport/data model, not by itself an integrity guarantee.
- External recording adds latency, storage and privacy obligations.
- Model/API telemetry alone is insufficient when local tool execution is authoritative; capture executor-side outcomes too.
- Integrity controls do not compensate for missing events, so measure capture completeness.

## Prototype experiment

For one unattended workflow, export model and tool events through a trusted recorder, keep recorder credentials outside the agent runtime, correlate executor outcomes with stable run IDs, apply deterministic redaction before storage, and reconstruct completed runs from control-plane telemetry alone.

Measure missing-event rate, recording latency, storage cost and reconstruction success.

## Sources

- GitHub, *OpenTelemetry in the GitHub Copilot app* (2026-09-22): https://github.blog/changelog/2026-09-22-opentelemetry-in-the-github-copilot-app/

## Related

- [Contain coding agents at the environment boundary first](environment-first-agent-containment.md)
- [Deterministic outer loop for coding-agent platforms](deterministic-outer-loop-agent-platform.md)
