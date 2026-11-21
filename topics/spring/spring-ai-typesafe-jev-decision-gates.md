# Spring AI DecisionModel for typed decisions and calibrated gates

## What it is

Spring AI **2.1.0 GA (2026-11-20)** includes the core DecisionModel interface for typed **YesNo**, **Choice**, and **Score** questions. It replaces the former Spring AI TypeSafe community integration described in this entry: that standalone integration is being archived, and related judge, refinement and guardrail patterns now live in **Spring AI Decision Extensions** (community-maintained, initially 0.5.0). This path is stable in Spring AI core, but extensions evolve on their own release schedule.

Use typed decisions when the output feeds a branch, score, triage category, document reranker or tool router rather than a paragraph.

## Use when

- Route/classify a request with known outcomes and a required confidence threshold.
- Decide *whether* any discovered tool or retrieved document applies.
- Risk-score content before applying non-security heuristics.
- Evaluate model responses against small independent criteria with a bounded retry/escalation path.

Never treat a model's YesNo score as a replacement for IAM, sandboxing, mandatory human approval or deterministic policy.

## Implementation (2.1 GA)

With a configured DecisionModel from a Spring AI starter, ask several narrow, explicitly described questions in one call:

~~~java
DecisionResponse response = decisionModel.call(
    DecisionPrompt.builder()
        .text(ticket)
        .question("defect", YesNoQuestion.builder()
            .text("Does the customer report a faulty product?")
            .whenYes("An item fails to work as specified")
            .whenNo("No product defect is described")
            .build())
        .question("satisfaction",
            ScoreQuestion.of("Customer satisfaction", "unhappy", "neutral", "happy"))
        .build());

if (!response.isRefused("defect") && response.getYesNo("defect").isYes(0.9)) {
    // Candidate for the warranty workflow, subject to normal policy.
}
if (!response.isRefused("satisfaction")) {
    int satisfactionLevel = response.getScore("satisfaction").getNearestLevel();
}
~~~

Implementations auto-configured by Boot starters:

- OpenAiDecisionModel under spring.ai.openai.decision.
- OllamaDecisionModel (local; default decision model d1-omni:600m).
- SystemOneDecisionModel via spring-ai-starter-model-systemone; compatible endpoints include TypeSafe/Jev and other System One implementations.

Prefer the official 2.1 documentation for **exact artifact names/provider setup**; don't reuse old org.springaicommunity TypeSafe 0.1.0 dependency snippets without a migration check.

### Composition

~~~text
trusted input and policy constraints
 -> typed decision(s) + probabilities
 -> calibrate against labeled data
 -> high-confidence: candidate route
 -> low-confidence: abstain / stronger model / human
 -> deterministic authorization and exact side-effect gate
~~~

For RAG, DecisionDocumentReranker and DecisionDocumentFilter are now in Spring AI core. DecisionToolIndex can reject the “none applies” case before selecting the closest wrong tool. Community DecisionJudge, DecisionSelfRefineAdvisor and DecisionGuardrailAdvisor require separate community artifacts; cap retries and mark refusals explicitly.

## Migration from Spring AI TypeSafe / Jev

The older 0.1.0 integration exposed TypeSafeClient.systemOne(), Noul/Choice/Score, JevJudge, JevSelfRefineAdvisor and JevGuardrailAdvisor. Do not assume class/package compatibility.

1. Move to Spring AI 2.1.0 GA and the Boot 4.2 baseline; review GA security fixes.
2. Replace custom TypeSafe calls with DecisionModel.call(DecisionPrompt) where possible.
3. Move judges, self-refinement and guardrails to the separately versioned Decision Extensions project only where needed.
4. Compare YesNo/Choice/Score outputs with a labeled holdout and re-tune thresholds.
5. Exercise low-confidence, never-applicable-tool, provider failure and malformed/unknown result cases.
6. Keep legacy adapters only behind an explicit deprecation boundary and remove the old community starter when migration is verified.

## Caveats

- Questions can be individually refused; check response.isRefused(name) before getYesNo/getScore/getChoice, or the typed accessor can throw IllegalStateException.
- Decision probabilities are not guaranteed calibrated across models, classes or workloads.
- Hosted providers have separate credentials/privacy/cost; local Ollama needs a compatible local model.
- A Choice implementation can still prefer a wrong known option; make no-match eligibility explicit.
- Decision Extensions is **community maintained**, not the same support/compatibility promise as Spring AI core.
- A high-confidence language-model score is not proof that access or execution is authorized.

## Prototype experiment

Collect 300 labeled routing cases including 20 adversarial “none of these tools” examples. Compare the legacy Jev path with 2.1 GA DecisionModel on two providers and a plain structured-chat baseline. Measure accuracy, Brier score/calibration, abstentions, p95 latency, cost and misrouted actions. Re-run after an extension-model upgrade; gate any new unsafe decision path with deterministic target authorization.

## Sources

- Spring AI 2.1 GA announcement and migration notice (2026-11-20): https://spring.io/blog/2026/11/20/spring-ai-2-1-0-GA-available-now/
- Original TypeSafe Jev integration (2026-09-21): https://spring.io/blog/2026/09/21/spring-ai-typesafe-structured-judgment/
- Spring AI Decision Models docs: https://docs.spring.io/spring-ai/reference/api/decision.html

## Related

- [Structured output validation](spring-ai-structured-output-validation.md)
- [Spring AI ToolSearchToolCallingAdvisor](spring-ai-tool-search-advisor.md)
- [Modern MCP caller scope](spring-ai-modern-mcp-client-scope.md)
