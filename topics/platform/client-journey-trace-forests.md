# Link client journeys to multiple server traces with a trace forest

## What it is

A user action or page view may create **several independent HTTP requests**, each with its own distributed `traceId`. Introduce an additional short-lived **journey/page-context ID** to group those request traces without forcing unrelated requests into one artificial trace. LinkedIn calls this `pageForestId` in its October 22, 2026 OpenTelemetry-based client/server platform.

The insight is two-level correlation: **journey ID answers “what did the user experience?”; trace ID answers “which backend path served one request?”**. This is separate from the [agent telemetry](../ai/external-authoritative-agent-telemetry.md) pattern, which tracks trusted tool effects.

## Use when

- investigating a UI event that fans out across independently traced API calls;
- correlating frontend/mobile failures with Java, Go or other backend services;
- adopting OpenTelemetry without replacing an existing client analytics transport;
- controlling cross-client/backend trace cost while keeping complete sampled journeys.

## Reusable instrumentation contract

~~~text
client starts page -> generate random page_context_id (expires on navigation)
  -> page root span + UI/render events
  -> HTTP request A: W3C traceparent + opaque page-context header
  -> HTTP request B: W3C traceparent + opaque page-context header
backend edge:
  -> extract traceparent and validated page-context
  -> start trace for this request and propagate through services
  -> tag server spans with page_context_id (bounded cardinality: traces only)
  -> export traces to OTel collector / queue / analysis store

query page_context_id -> all request trace IDs -> per-request span waterfall
~~~

Do not reuse the page ID as the `traceId`. Follow the normal W3C Trace Context propagation and keep the extra ID **opaque, random, time-bounded, and privacy reviewed**. In practice, the field may be a specific allowlisted HTTP header rather than public W3C baggage; if baggage is used, account for propagation to third parties and payload/privacy risks.

A pseudocode request interceptor sketch:

~~~typescript
const pageContextId = crypto.randomUUID(); // rotate at page navigation
function instrumentedFetch(url: string, options: RequestInit) {
  const headers = new Headers(options.headers);
  headers.set("X-Page-Context", pageContextId);
  // The OpenTelemetry browser instrumentation is responsible for traceparent.
  // Never insert private user identifiers or bearer tokens into these headers.
  return fetch(url, { ...options, headers });
}
~~~

On the Java gateway, validate that the header fits an allowlisted format/length and has no cross-origin trust significance. Add it as a span attribute, then propagate via an explicit internal context mechanism. **Do not use a client-controlled correlation header for authorization**.

### Sampling and ingestion design

- Make a **coordinated head-sampling choice** so a selected journey does not retain only its frontend half. LinkedIn uses a pseudonymized member key and propagates the decision; for a smaller product, a stable random session hash or journey ID and OTel sampler rules may be simpler. Do not introduce stable cross-visit tracking merely for telemetry.
- Protect quality-of-experience (QoE) signals such as failed actions/slow starts from aggressive debug sampling. Make sampling tiers and their purpose visible.
- Provide a client span processor with bounded queue, batch size, per-trace limits and a **best-effort background/termination flush**; abrupt OS termination still loses data.
- Reuse an existing mobile/web analytics delivery SDK through a transformation exporter when replacing transport is too risky. LinkedIn uses its tracking pipeline for clients and Kafka for server spans, then ClickHouse for storage.
- Index by `(page_context_id, trace_id, timestamp)` or equivalent. Compare query latency and completeness for both journey view and trace drill-down.
- Redact/allowlist attributes on SDK and ingestion boundaries; restrict access and retention. **Never log raw user content, tokens or private page data** as trace attributes.

## Pitfalls and tests

- Independent client/server sampling can produce **misleading half-traces**. Validate end-to-end propagation with a forced-sampling canary.
- Long-lived page-root spans can remain open indefinitely. LinkedIn closes the root early and derives effective page duration from descendants; document any derived semantics so dashboards are interpretable.
- One trace ID does not capture multiple independent requests: correlate with journey ID, but avoid treating all requests as parent-child when they are siblings.
- Span cardinality, high-frequency UI events and export retries can overwhelm mobile devices and collectors; throttle/debug-gate where appropriate.
- Clock skew and dropped client spans can confuse waterfall timing; include quality indicators, clocks and a “partial journey” flag.
- A session-level sampling key may create privacy/fairness trade-offs; evaluate retention, pseudonymization, bias and opt-out.

## Evidence and prototype

LinkedIn reports end-to-end client–server debugging across iOS, Android, web and Java/Python/Go/Rust backends, with Kafka/ClickHouse ingestion. It reports ~30% of steady-state client spans as non-metric-critical and estimates **up to** ~90% of backend span volume as debug-tier candidates; this is **not** a measured generic savings rate. Published case studies use illustrative trace data and do not provide a controlled mean-time-to-resolution comparison.

Instrument a test UI with three parallel requests, one deliberately failing, and a Java backend with one downstream call. In a test collector verify all three request traces share the same page context; no raw PII is transmitted; a selected journey is sampled coherently; one dropped client span is visibly marked incomplete. Then measure query latency, exporter overhead, capture ratio after simulated mobile backgrounding and ingestion cost per 10,000 journeys.

## Sources

- LinkedIn Engineering, *From Member Tap to Backend: How LinkedIn Traced the Member Journey with OpenTelemetry* (2026-10-22): https://www.linkedin.com/blog/engineering/infrastructure/from-membertap-to-backend-how-linkedin-traced-the-member-journey-with-opentelemetry
- InfoQ reporting (2026-11-03): https://www.infoq.com/news/2026/11/linkedin-opentelemetry-journey/
- OpenTelemetry trace model: https://opentelemetry.io/docs/concepts/signals/traces/

## Related

- [External authoritative agent telemetry](../ai/external-authoritative-agent-telemetry.md)
- [OpenTelemetry Java agent 3 migration](opentelemetry-java-agent-3-preview-migration.md)
