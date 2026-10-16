# Safely preview OpenTelemetry Java agent 3.0

## What it is

The Java agent 2.32.0 release provides a preview of the upcoming 3.0 defaults. Semantic convention changes affect JDBC and messaging traces, database connection pool metrics, logs, and dashboards. Treat the migration as an observability-contract change, not just an agent upgrade.

## Use when

Upgrading Java/Spring services instrumented with the OpenTelemetry Java agent, particularly services using JDBC, Kafka, RabbitMQ, structured logs, or dashboards that group by old database and messaging attributes.

## Implementation workflow

1. Pin the Java agent at 2.32.0 for a staging deployment; capture a baseline of spans, metrics, logs, dashboard queries, and alert thresholds.
2. Preview new conventions while leaving the combined preview disabled:

```bash
OTEL_INSTRUMENTATION_COMMON_V3_PREVIEW=false
OTEL_SEMCONV_STABILITY_OPT_IN=database/dup,code/dup
OTEL_SEMCONV_STABILITY_PREVIEW=messaging/dup
```

3. Compare emitted old and new attributes: `db.system` becomes `db.system.name`; `db.name` becomes `db.namespace` (may include schema); `db.statement` becomes `db.query.text`. Pool metric names and duration units can change.
4. Exercise producer-consumer traces and assess parent/link changes. A Kafka process span may become the child of an active application span, with a link to the producer instead of being its child.
5. Test the combined preview using `OTEL_INSTRUMENTATION_COMMON_V3_PREVIEW=true`. It changes defaults, and does not retain dual emission in the affected domains.
6. Review structured log attributes and data minimization; some key-value fields appear by default with the preview. The common structured-attributes include/exclude selectors replace older source-specific settings.
7. Compare actual alerts, units and query results. Test any Zipkin exporter, Hibernate instrumentation and custom Java agent extensions; their preview defaults can change. Promote to production only after canary and explicit rollback validation.

## Caveats

Dual emission does not reproduce both span names/kinds, nor every legacy connection pool metric. Some dimensions change value as well as name: `db.namespace` may contain database plus schema. A metric duration that changes from milliseconds to seconds requires threshold conversion. The 3.0 release was not yet generally available when this entry was written (2026-10-16); repeat all tests on the actual GA build.

## Experiment

Replay JDBC and messaging traffic in one Spring Boot staging service with the baseline, dual-emission and umbrella-preview configurations. Save telemetry samples and check alerts, trace parentage, dimensions, log fields, agent startup and runtime overhead. Use an allowlisted canary rollout for promotion.

## Sources

- https://opentelemetry.io/blog/2026/java-agent-3.0-preview/
- https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/tag/v2.32.0

## Related

- [External authoritative agent telemetry](../ai/external-authoritative-agent-telemetry.md)
