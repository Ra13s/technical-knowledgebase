# Record/replay AI model calls with request-signed cassettes

## What it is

A deterministic integration-test pattern for applications that call chat or embedding models: record a real provider response once, store it as a cassette, and replay it in ordinary CI runs without network access or provider credentials.

The important part is **request matching**. A cassette should only replay when the semantically relevant request still matches. If prompt messages, model options, tool definitions, provider parameters or embedding input change, the fixture must be treated as stale rather than silently returning an unrelated old response.

Vectors VCR implements this pattern for Spring AI and LangChain4j with JUnit 5/TestNG support.

## Use when

Use record/replay tests when you need deterministic tests around your application's AI integration layer, for example:

- prompt/template regressions;
- structured-output parsing;
- tool-call handling;
- streaming response assembly;
- provider metadata/token-usage handling;
- embeddings and downstream retrieval plumbing;
- CI environments where API keys or network calls are undesirable.

Do **not** use cassette playback as proof that the current model still produces a good answer. That requires live evals.

## Recommended workflow

Split development and CI behavior deliberately:

```text
local development / explicit refresh
  -> real provider
  -> record or update cassette

normal CI
  -> strict playback
  -> fail on missing/stale cassette
```

For Vectors VCR, the core modes are:

- `PLAYBACK` — replay only an exact matching cassette; fail if missing/stale;
- `RECORD` — always call the real model and replace/store the response;
- `RECORD_NEW` — record only when no cassette exists;
- `RECORD_FAILED` — re-record tests that previously failed;
- `PLAYBACK_OR_RECORD` — replay exact matches, otherwise call the provider and refresh;
- `OFF` — bypass the harness.

Prefer strict `PLAYBACK` in CI so a prompt change cannot quietly trigger a paid/network call.

## Example with Vectors VCR

Representative dependencies for Vectors 0.1.21:

```text
com.integrallis:vectors-vcr-junit5:0.1.21
com.integrallis:vectors-vcr-serde-avaje:0.1.21   # or Jackson
com.integrallis:vectors-vcr-spring-ai:0.1.21     # Spring AI
com.integrallis:vectors-vcr-langchain4j:0.1.21   # LangChain4j
```

A test can opt into record/replay behavior:

```java
@VCRTest(
    mode = VCRMode.PLAYBACK_OR_RECORD,
    dataDir = "src/test/resources/vcr-data"
)
class ProductAssistantTest {

    @VCRModel
    ChatModel chatModel;

    @Test
    void extractsProductDecision() {
        // production-style model call through the wrapped model
    }
}
```

Use the library's current API names/version from its documentation rather than copying this example indefinitely.

## Implementation rules

### 1. Sign the complete request, not only the prompt text

A useful cassette key/signature includes all inputs that can materially alter the provider response.

For chat calls this typically means:

```text
messages
+ model identity/label
+ generation/options
+ provider-specific parameters
+ tool definitions
+ model defaults that affect the request
```

For embeddings include at least the input and model/options.

If the signature changes, fail as stale in strict playback. Vectors raises a stale-cassette error for this case.

### 2. Preserve the response shape your code actually consumes

Do not reduce a recorded response to final text if production code relies on more.

Record/replay should preserve as applicable:

- multiple generations;
- assistant/tool-call messages;
- streaming chunk order;
- structured output;
- token usage;
- rate-limit/provider metadata;
- response attributes and filtering metadata;
- embedding vectors.

Otherwise the test may pass while exercising a simplified fake that does not resemble the integration contract.

### 3. Never keep partial fixtures from failed recordings

If a test fails while a new cassette is being recorded, discard the newly written fixture or mark it failed. A half-recorded interaction can poison later playback and make a broken run look deterministic.

Vectors VCR removes newly written cassettes for failed recording tests and tracks failure state in its registry.

### 4. Make fixture refresh explicit in review

Treat cassette updates similarly to snapshot/golden-file changes:

```text
prompt/tool/model contract changed
  -> intentionally run record/update mode
  -> inspect cassette diff
  -> commit fixture with code/test change
```

Do not put `PLAYBACK_OR_RECORD` into an unrestricted CI environment where missing fixtures can silently call external providers.

### 5. Scrub secrets and sensitive data

Cassettes can contain prompts, user content, tool arguments, provider metadata and model output. Before committing them:

- remove credentials/tokens;
- avoid real customer/private data;
- redact environment-specific identifiers when they are not part of the contract;
- set repository retention/access rules appropriate to the recorded content.

### 6. Keep live evals as a separate lane

Record/replay answers a different question from an eval:

```text
VCR test:  "Does our application handle this known provider interaction correctly?"
live eval: "Does today's model/provider still behave well on this task?"
```

A frozen cassette can continue passing even after the provider model has regressed or improved. Run periodic or release-gated live evals for behavioral quality.

## Why it is useful

Mocking an AI client by hand often creates a toy response that omits tool calls, metadata, streaming details or provider-specific behavior. Calling the live provider in every unit/integration test is slow, nondeterministic, expensive and requires secrets.

Request-signed cassettes provide a useful middle layer: realistic recorded contracts with deterministic local/CI playback.

## Failure modes and caveats

- **False confidence:** playback verifies application integration, not current model quality.
- **Fixture drift:** permissive refresh modes can hide prompt or request changes; CI should fail stale fixtures.
- **Sensitive recordings:** model interactions may contain data that should not enter Git.
- **Provider evolution:** a cassette cannot reveal a breaking provider behavior change until you refresh or run live tests.
- **Oversized fixtures:** large streaming/tool traces can become noisy; keep scenarios focused.
- **Framework coupling:** wrapper libraries must preserve the Spring AI/LangChain4j semantics your application actually uses.

## Prototype experiment

Pick 5-10 AI integration tests that currently depend on a live provider:

1. record cassettes from the real model;
2. switch ordinary CI to strict playback;
3. deliberately change one prompt option and verify CI fails with a stale-cassette signal;
4. exercise at least one tool-call and one streaming scenario;
5. fail a recording run intentionally and verify no partial fixture survives;
6. compare runtime, flakiness, secret requirements and debugging quality with the live-only suite;
7. keep a small separate live-eval job to catch provider/model behavior changes.

## Sources

- https://integrallis.github.io/vectors/docs/vectors/current/testing.html

This source was first logged as an experiment candidate in the 2026-09-12 intake and is promoted here after a dedicated testing-focused review rather than treated as a newly discovered source.

## Related

- [Spring AI structured output validation and self-correction](../spring/spring-ai-structured-output-validation.md)
- [Microcks Testcontainers contract testing](microcks-testcontainers-contract-tests.md)
