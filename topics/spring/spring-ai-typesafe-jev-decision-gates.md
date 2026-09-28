# Spring AI TypeSafe Jev for typed decision gates

## What it is

A Spring AI Community integration for TypeSafe AI's hosted Jev decision model. Jev does not generate prose; it evaluates supplied state with typed questions and returns numeric truth values, categorical probabilities or rubric scores.

Use it when the result should feed code branching, thresholds or validation rather than become user-facing text.

Version referenced here: `org.springaicommunity:*:0.1.0` (September 2026).

## Use when

Good fits include:

- route or classify a request;
- score quality against explicit criteria;
- gate an expensive model call;
- filter or rerank retrieved documents;
- decide whether any tool applies, then select one;
- input/output screening;
- bounded self-refinement after a failed typed check.

Keep a generative chat model for drafting, explaining and open-ended reasoning.

## Setup

```xml
<dependency>
  <groupId>org.springaicommunity</groupId>
  <artifactId>spring-ai-starter-typesafe</artifactId>
  <version>0.1.0</version>
</dependency>

<dependency>
  <groupId>org.springaicommunity</groupId>
  <artifactId>typesafe-spring-ai</artifactId>
  <version>0.1.0</version>
</dependency>
```

```properties
spring.ai.typesafe.api-key=${TYPESAFE_API_KEY}
```

The starter provides a `TypeSafeClient`.

## Three decision primitives

```java
SystemOneResponse response = typeSafeClient.systemOne(
    ticket,
    Map.of(
        "urgent", Noul.of("Does this convey urgency?"),
        "team", Choice.builder()
            .instructions("Which team should handle this?")
            .option("billing", "Payments, invoicing, refunds")
            .option("technical", "Bugs, outages, integrations")
            .option("sales", "Pricing and upgrades")
            .build(),
        "frustration", Score.of(
            "How frustrated is the customer?",
            "Calm", "Frustrated", "Very angry")
    )
);

double urgent = response.noulValue("urgent");
String team = response.choiceValue("team");
double confidence = response.choice("team").confidence();
double frustration = response.scoreValue("frustration");
```

The primitives are:

- `Noul`: yes/no judgement as a value in [0,1];
- `Choice`: one label plus option probabilities/confidence;
- `Score`: continuous position on an ordered rubric plus confidence.

Prefer several narrow questions with explicit descriptions over one vague judgement.

## Compose thresholds in code

```java
JevJudge judge = JevJudge.builder(typeSafeClient)
    .score("helpfulness", helpfulnessRubric, 2.0)
    .noul("is_plausible", plausible, 0.8)
    .noul("is_grounded", grounded, 0.8)
    .build();

JevVerdict verdict = judge.judge(question, answer);
```

Treat `INCONCLUSIVE` separately from pass/fail. Low confidence is a routing signal for stronger evaluation or human review, not proof that the answer is wrong.

## Bounded self-refinement

```java
ChatClient chatClient = ChatClient.builder(chatModel)
    .defaultAdvisors(
        JevSelfRefineAdvisor.builder()
            .judge(judge)
            .maxRepeatAttempts(3)
            .failOnExhaustedAttempts(true)
            .build())
    .build();
```

Workflow:

```text
generate
  -> typed judgement
      -> pass: return
      -> fail: convert failed criteria to feedback
               -> retry from original prompt
  -> stop after bounded attempts
```

Use `failOnExhaustedAttempts(true)` when downstream code must not silently consume the advisor's best effort after the retry budget is exhausted.

## Screening shape

`JevGuardrailAdvisor` can screen both input and final output. Unlike self-refinement, a blocked decision should terminate rather than trigger repeated generation.

Keep hard authorization, sandboxing and side-effect controls deterministic. A probabilistic decision model is a classifier/judge, not an authorization boundary.

## Spring AI integration points

The project integrates with existing Spring AI SPIs:

- `CallAdvisor`: self-refine and screening advisors;
- `DocumentPostProcessor`: document filtering/reranking;
- `ToolIndex`: tool selection;
- `Evaluator`: evaluation.

For tool selection, remember that a `Choice` always returns a winner. Ask a separate `Noul` such as "does any tool apply?" before choosing among tools.

## Why it is useful

Many AI pipelines spend a full generative model call to get a small decision and then parse prose or JSON back into code. A typed decision model makes the output shape and confidence explicit and can be composed with ordinary Java thresholds.

> If the next step is a branch, score or classification rather than text generation, evaluate a typed decision primitive before reaching for another chat completion.

## Caveats

- Version 0.1.0 is early and APIs can change.
- Jev is a hosted external service and requires a TypeSafe API key.
- It does not stream.
- State must serialize as string/object/array/null; bare number/boolean state is rejected.
- Per-document evaluation is one call per document; filter before reranking large sets.
- `Choice` always chooses one option unless "none" is separately modelled.
- Thresholds and confidence need calibration on your own labelled tasks.
- TypeSafe's published performance comparisons are vendor benchmarks. Measure production latency and cost in your environment.

## Prototype experiment

Take one current Spring AI classifier/evaluator implemented with a chat model:

1. collect 100-500 labelled examples;
2. express the decision as narrow `Noul`, `Choice` or `Score` questions;
3. tune thresholds on a held-out set;
4. compare quality/calibration, p50/p95 latency and cost with the current approach;
5. route low-confidence/`INCONCLUSIVE` cases to the existing stronger path;
6. replace the existing path only if the cascade preserves required quality.

## Sources

- Spring, *Spring AI and TypeSafe Jev: Fast, Cheap, Structured Decisions* (2026-09-21): https://spring.io/blog/2026/09/21/spring-ai-typesafe-structured-judgment/

## Related

- [Spring AI structured output validation and self-correction](spring-ai-structured-output-validation.md)
- [Spring AI ToolSearchToolCallingAdvisor](spring-ai-tool-search-advisor.md)
