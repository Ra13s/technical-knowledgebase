# API mocks and contract conformance with Microcks Testcontainers

## What it is

Microcks can consume API contracts such as OpenAPI and AsyncAPI, generate live mocks from their examples, and run conformance tests against a real implementation. Its Java Testcontainers module lets the same workflow run inside JUnit without a shared Microcks environment.

## Use when

Use this when an API specification should be an executable development artifact rather than documentation only, especially when:

- a consumer must develop before the provider is available;
- teams maintain OpenAPI/AsyncAPI contracts and want mocks derived from the same source;
- CI should detect when an implementation stops conforming to its contract;
- handwritten mocks are drifting from the API specification.

## Start Microcks from a contract

Add the Testcontainers module using a version compatible with the Microcks image used by the project:

```xml
<dependency>
  <groupId>io.github.microcks</groupId>
  <artifactId>microcks-testcontainers</artifactId>
  <scope>test</scope>
</dependency>
```

Then start an ephemeral Microcks instance and import the contract as the main artifact:

```java
MicrocksContainer microcks = new MicrocksContainer(
    DockerImageName.parse("quay.io/microcks/microcks-uber:1.14.0"))
    .withMainArtifacts("orders-openapi.yaml");

microcks.start();
```

Pin the image and library versions in real builds rather than copying `latest` examples.

## Use the generated mock in consumer tests

Microcks creates mock endpoints from examples in the imported contract:

```java
String baseUrl = microcks.getRestMockEndpoint("Orders API", "1.0.0");

HttpResponse<String> response = httpClient.send(
    HttpRequest.newBuilder(URI.create(baseUrl + "/orders/123"))
        .GET()
        .build(),
    HttpResponse.BodyHandlers.ofString());

assertEquals(200, response.statusCode());
```

For interaction-style tests, also verify that the mock was invoked:

```java
microcks.verify("Orders API", "1.0.0");
```

This is preferable to maintaining a second handwritten mock contract when the OpenAPI examples already express the expected interaction.

## Run the same contract against the implementation

If the service under test runs on the test host, expose its port before Microcks starts:

```java
Testcontainers.exposeHostPorts(port);
microcks.start();
```

Then launch an OpenAPI schema conformance test:

```java
TestRequest request = new TestRequest.Builder()
    .serviceId("Orders API:1.0.0")
    .runnerType(TestRunnerType.OPEN_API_SCHEMA.name())
    .testEndpoint("http://host.testcontainers.internal:" + port)
    .filteredOperations(List.of("GET /orders/{id}"))
    .timeout(Duration.ofSeconds(3))
    .build();

TestResult result = microcks.testEndpoint(request);
Assertions.assertSuccess(result);
```

Use `filteredOperations(...)` when a component test intentionally implements only part of a larger contract. In broader CI tests, omit the filter so the full contract is exercised.

## Decision rule

Use **one API artifact for both directions** whenever possible:

```text
OpenAPI / AsyncAPI contract
        |                 |
        v                 v
consumer mock       provider conformance test
```

That gives contract examples and schemas one place to evolve, instead of allowing mocks and provider tests to encode independent versions of the API.

## Why it is useful

- Consumers can work against generated mocks before the provider exists.
- The provider can be checked against the exact same contract artifact.
- Contract drift becomes a failing test instead of a documentation problem.
- The Testcontainers form works locally and in CI without maintaining a shared test service.
- Microcks also supports SOAP, GraphQL, gRPC and AsyncAPI-based event APIs; the exact runner/setup differs by protocol.

## Caveats / when not to use

- Contract conformance proves schema/example compatibility, not business correctness.
- Examples in the contract need to be realistic enough to produce useful mocks.
- Keep compatibility between the Microcks image and Testcontainers module under dependency management.
- Testcontainers requires a container runtime in the development/CI environment.
- Do not let a very broad contract test replace smaller application tests; use it as an API-boundary check.

## Sources

- Microcks Testcontainers guide: https://microcks.io/documentation/guides/usage/developing-testcontainers/
- Microcks Testcontainers capability matrix: https://microcks.io/documentation/references/testcontainers-modules/
- Concrete Java example: https://www.baeldung.com/java-microks-api-mocking-testing
