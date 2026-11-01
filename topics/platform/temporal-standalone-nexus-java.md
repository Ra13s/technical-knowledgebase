# Start durable cross-service Nexus operations directly from Java

## What it is

**Temporal Standalone Nexus Operations** let a normal Java service, HTTP controller, BFF or script invoke a Nexus service operation without running a caller Workflow. The operation's durable lifecycle belongs to Temporal. The handler-side Nexus Service, operation contract and Worker remain the same as for calls originating inside a Temporal Workflow; only the caller changes.

Choose Standalone Nexus for **one durable cross-service operation** (e.g. collect payment, generate export, ingest a document). Keep a caller Workflow when you need to orchestrate multiple operations, coordinate compensation, or maintain business state between calls. This is distinct from simply making an HTTP request and retrying it.

## Compatibility and status (2026-11-01)

- Standalone Nexus is **Public Preview** in Temporal Cloud; the Java docs label the SDK surface **Pre-release / experimental**.
- Requires **Temporal Java SDK 1.36.1+** and **Temporal CLI 1.9.0+** for local development.
- At-least-once operation execution is possible. A durable request record is **not** exactly-once external side effect execution. The downstream payment/export handler still needs an idempotency key.
- Public Preview lacks some reset, batch actions, UI bulk controls and selected HA/multi-region capabilities. Recheck compatibility and runtime requirements before production rollout.

## Use when

- a Spring Boot endpoint must call a shared service asynchronously across a Namespace boundary;
- a client may disconnect before the durable operation finishes;
- a dedicated caller Workflow that only proxies one Nexus call adds unnecessary complexity;
- multiple applications or agents should call the same strongly typed service operation;
- application state can persist the Nexus operation ID, then retrieve its outcome on another request.

## Runnable Java recipe

Start a Temporal CLI 1.9+ dev server and a handler worker:

~~~sh
temporal server start-dev \
  --namespace my-caller-namespace \
  --namespace my-handler-namespace

temporal operator nexus endpoint create \
  --name my-nexus-endpoint \
  --target-namespace my-handler-namespace \
  --target-task-queue nexus-handler-queue

# In the temporalio/samples-java checkout:
TEMPORAL_NAMESPACE=my-handler-namespace \
  ./gradlew -q :core:execute -PmainClass=io.temporal.samples.nexusstandalone.handler.HandlerWorker

TEMPORAL_NAMESPACE=my-caller-namespace \
  ./gradlew -q :core:execute -PmainClass=io.temporal.samples.nexusstandalone.StandaloneClientStarter
~~~

The Java SDK's typed client API (imports and fully configured `stubs`/`options` are in the linked sample):

~~~java
NexusClient nexusClient = NexusClient.newInstance(stubs, options);
NexusServiceClient<GreetingNexusService> service =
    nexusClient.newNexusServiceClient(
        GreetingNexusService.class, "my-nexus-endpoint");

// Caller must provide a stable operation ID; the SDK does not generate one.
StartNexusOperationOptions opOptions =
    StartNexusOperationOptions.newBuilder()
        .setId("export-" + exportRequestId)
        .setScheduleToCloseTimeout(Duration.ofMinutes(5))
        .build();

NexusOperationHandle<GreetingOutput> handle =
    service.start(GreetingNexusService::startGreeting,
                  opOptions, new GreetingInput("World"));
// Store exportRequestId / operation ID in the application database.
// Await (or retrieve) the result later, outside the initial HTTP request:
GreetingOutput output = handle.getResult();
~~~

For a synchronous caller, `service.execute(...)` blocks until completion, and `executeAsync(...)` returns a `CompletableFuture`. Both require a caller-provided ID. Use the operation handle's `getResult()`, `getResultAsync()` or timed wait; handle `NexusOperationException` distinctly from a transient transport error.

Use the CLI to observe the authoritative operation state:

~~~sh
temporal nexus operation list --namespace my-caller-namespace \
  --query 'Endpoint = "my-nexus-endpoint"'
temporal nexus operation result \
  --namespace my-caller-namespace --operation-id my-greet-op
~~~

`NexusClient.listNexusOperationExecutions(query)` and `countNexusOperationExecutions(query)` expose operational inspection APIs. An Operation ID is a critical part of the business protocol: use an application-stable ID so retries and client disconnects do not create duplicate business operations.

## Recommended HTTP-service state model

~~~text
POST /exports (client supplies/reuses idempotency key)
  -> validate identity + payload
  -> persist request(id, operationId, PENDING) transactionally
  -> launch Nexus Operation with stable operationId
  -> 202 Accepted + request ID

GET /exports/{id}
  -> verify ownership
  -> lookup operationId
  -> query/resume operation result / status
  -> map to PENDING | COMPLETE | FAILED
~~~

Be careful at the DB-write / Nexus-start boundary: there is no distributed transaction between them. Recover PENDING records via authoritative operation lookup/restart with the *same* stable ID and idempotent handler, rather than blindly starting another differently identified operation.

## Tradeoffs and failure modes

- Durable acceptance, retries and stored execution state are handled by Temporal, but business side effects need handler-level idempotency. Avoid an unbounded web request waiting for remote completion.
- The caller Namespace and handler Namespace, Nexus endpoint, task queue, authentication and network policies must agree.
- A single Nexus call is a good fit; if business logic branches into several steps with compensation, use a real Workflow.
- Avoid treating preview APIs as stable: pin the Java SDK, test upgrade/downgrade and confirm the server/CLI compatibility matrix.
- Correlate app request ID, Nexus operation ID, Temporal workflow/activity IDs and failure cause in telemetry. Count PENDING age, retry attempts and post-acceptance cancellations.

## Experiment

Build a two-service Java 17+ test using `temporalio/samples-java`. Begin an operation from an HTTP endpoint, kill the caller process immediately after acceptance, restart it and retrieve the same operation's result. Separately kill the handler worker and verify retry/recovery, then simulate duplicate POSTs and assert one business effect. Measure p95 completion latency, ambiguity recovery time and duplicate-effect count. Repeat with one dedicated proxy Workflow and document the operational/cost tradeoff.

## Sources

- https://temporal.io/blog/standalone-nexus-operations
- https://docs.temporal.io/standalone-nexus-operation
- https://docs.temporal.io/develop/java/nexus/standalone-operations
- https://github.com/temporalio/samples-java

## Related

- [Durable agent tool effects and fencing](../ai/durable-agent-tool-effects-and-fencing.md)
- [Idempotent agent tool side effects](../ai/idempotent-agent-tool-side-effects.md)
- [Durable document ingestion](durable-document-ingestion-pipeline.md)
