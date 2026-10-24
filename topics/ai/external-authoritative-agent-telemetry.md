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

## Correlate agent, model gateway and executor spans

AWS's October 21, 2026 EKS example gives a runnable **telemetry transport** for the existing authoritative-recording rule. Instrument agent SDK calls and the LLM gateway with OpenTelemetry, propagate W3C `traceparent` through the HTTP client into the gateway, and preserve `agent.name`, model, tool, token usage, latency and per-operation cost. The two approaches differ in *who owns export*:

- **Direct agent/gateway -> Langfuse OTLP:** minimal moving parts for a small system, but backend, sampling and redaction settings are replicated across agents.
- **Agent/gateway -> shared OTel Collector -> backend:** central batching, redaction, tail sampling and metrics export, at the cost of another production dependency. Use memory limits, bounded retry queues, replicated collectors and monitored dropped-span counters.

Portable collector sketch (choose the actual exporter/auth settings for the backend):

~~~yaml
receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318
processors:
  batch:
    timeout: 5s
    send_batch_size: 256
exporters:
  otlphttp/observability:
    endpoint: ${env:OTLP_BACKEND_ENDPOINT}
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlphttp/observability]
~~~

Set `OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318` in agent and gateway deployments, and instrument the HTTP client so `traceparent` reaches the gateway. AWS publishes a full ArgoCD/EKS/Bifrost/Langfuse sample: https://aws.amazon.com/blogs/containers/observability-patterns-for-ai-agents-on-amazon-eks/ (2026-10-21).

**Critical distinction:** a model-generated span or SDK telemetry is not independent proof that a tool executed. Keep gateway authorization decisions, executor-side results and durable write receipts in **trusted control-plane** telemetry; correlate them with the agent trace by run and tool-call ID. A shared collector provides a convenient export policy, **not** immutable audit evidence.

### Practical regression check

For one agent request that calls a tool and one that triggers model fallback, assert: the agent root, model-gateway span and tool executor are linked to the same run; actual provider/model and token usage are attributable; a denied tool call is visible; a fabricated agent "success" cannot erase the executor's failure. Under exporter outage, measure dropped events, queue age and retry; keep a protected audit stream for indispensable decision records even when sampling ordinary traces. Redact sensitive prompts before storage, verify that sampling keeps high-impact failures, and cap high-cardinality labels and telemetry spend.

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
- AWS Containers, *Observability patterns for AI agents on EKS* (2026-10-21): https://aws.amazon.com/blogs/containers/observability-patterns-for-ai-agents-on-amazon-eks/

## Related

- [Contain coding agents at the environment boundary first](environment-first-agent-containment.md)
- [Deterministic outer loop for coding-agent platforms](deterministic-outer-loop-agent-platform.md)
