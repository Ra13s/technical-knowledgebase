# Spring AI structured output validation and self-correction

## What it is

Spring AI 2.0 can combine provider-native structured output with response-side JSON Schema validation and automatic retry.

The two controls solve different problems:

- `useProviderStructuredOutput()` asks a supporting model/provider to enforce the schema at generation time.
- `validateSchema()` validates the completed response in Spring AI and retries with the validation error when the output is malformed.

## Use when

Use this for LLM output that downstream code treats as data rather than prose, for example:

- routing/classification results;
- extraction into Java records/classes;
- persisted AI-generated metadata;
- workflow decisions;
- tool-independent DTO generation.

If malformed output would otherwise become an exception, bad persisted data, or a wrong application branch, add validation instead of relying only on prompt wording.

## How to use it

For a typed response where correctness matters:

```java
ActorsFilms films = chatClient.prompt()
    .user("Generate the filmography for a random actor.")
    .call()
    .entity(ActorsFilms.class, spec -> spec
        .useProviderStructuredOutput()
        .validateSchema());
```

Use only `validateSchema()` when provider-native structured output is unavailable or you want the portable prompt-based path:

```java
ActorsFilms films = chatClient.prompt()
    .user("Generate the filmography for a random actor.")
    .call()
    .entity(ActorsFilms.class, spec -> spec.validateSchema());
```

On validation failure, Spring AI appends the concrete schema error to the user prompt and re-runs the model. The default maximum is three attempts.

For explicit retry control, register a `StructuredOutputValidationAdvisor`:

```java
var validationAdvisor = StructuredOutputValidationAdvisor.builder()
    .outputType(ActorsFilms.class)
    .maxRepeatAttempts(2)
    .build();

ChatClient chatClient = ChatClient.builder(chatModel)
    .defaultAdvisors(validationAdvisor)
    .build();
```

You can also provide a JSON Schema directly with `outputJsonSchema(...)` instead of deriving it from a Java type; the two builder options are mutually exclusive.

## Decision rule

For structured LLM output:

1. Start with `.entity(Type.class)` for low-risk/best-effort conversion.
2. Add `.useProviderStructuredOutput()` when the selected provider/model supports native schema enforcement.
3. Add `.validateSchema()` when a malformed response must be automatically caught and corrected.
4. Use both for important typed outputs where provider enforcement plus an application-side safety net is worth the extra retry cost.
5. Put a small retry ceiling on high-volume paths and measure cumulative token/latency cost.

## Why it is useful

- Turns malformed structured output into a bounded self-correction loop instead of ad-hoc retry code.
- Retry prompts contain the actual schema failure, so retries are targeted rather than blind.
- Provider-native mode removes JSON formatting instructions from the prompt and uses the provider's structured-output API.
- Spring AI reports cumulative token usage across validation attempts, which makes the retry cost observable.

## Caveats / when not to use

- `validateSchema()` requires the complete response; streaming structured output is not supported.
- Retries consume extra tokens and latency. Do not silently raise retry counts without measuring failure rate and cost.
- Provider-native structured output support and limitations vary by provider/model. Test the exact model version used in production.
- If you pass a custom `StructuredOutputConverter`, implement `getJsonSchema()` if you expect native structured output or schema validation to work; without a schema those features cannot enforce the custom format.
- Tool calling already has structured tool arguments/results; this recipe is for normal model responses converted into application types.

## Version / compatibility

The APIs described here are available in Spring AI 2.0.x. The current reference documentation is 2.0.1. Provider-native support is model-specific even when the Spring AI API is portable.

## Sources

- https://spring.io/blog/2026/06/23/spring-ai-self-correcting-structured-output/
- https://docs.spring.io/spring-ai/reference/api/structured-output/validation.html
- https://docs.spring.io/spring-ai/reference/api/structured-output/native.html
- https://docs.spring.io/spring-ai/reference/api/structured-output/converters.html

## Related

- [`Spring AI ToolSearchToolCallingAdvisor`](spring-ai-tool-search-advisor.md) — progressively disclose large tool catalogs; structured output validation addresses typed model responses instead.