# Precision-first multi-stage AI code review

## What it is

A code-review agent architecture that treats review as a pipeline rather than one prompt: specialized issue generators produce candidates, a separate grader scores them, semantic deduplication collapses overlaps, and low-value categories are suppressed using production feedback.

Uber's uReview uses this shape at large scale and reports sustained usefulness above 75% for posted comments.

## Use when

Use this when AI code review is producing too many false positives, duplicate comments, style nits, or low-value suggestions that developers ignore.

It is especially appropriate when review volume is high enough that a single generic reviewer becomes noisy and hard to tune.

## Pipeline

```text
changed files
  -> eligibility filter
  -> specialized reviewers
       -> correctness
       -> best practices
       -> security
  -> independent comment grader
  -> confidence thresholds
  -> semantic dedupe
  -> category/value filter
  -> inline comments
  -> developer feedback + addressed/not-addressed telemetry
```

## Implementation rules

### 1. Filter low-signal inputs before the model

Exclude generated code, known experimental directories and other file classes where review comments are rarely useful.

### 2. Split review by concern

Use separate prompts/context for issue classes that need different evidence or thresholds. Examples:

- correctness / logic
- exception/error handling
- security
- organization-specific best practices

Do not assume one prompt is optimal for all categories.

### 3. Grade every candidate comment separately

A second model/prompt should judge whether the proposed comment is correct and worth interrupting a developer for.

Store metadata such as:

```json
{
  "assistant": "security",
  "language": "java",
  "category": "correctness:null-check",
  "confidence": 0.91
}
```

Tune thresholds per language/category/reviewer when enough data exists; a single global threshold is usually too crude.

### 4. Deduplicate semantically

Multiple reviewers can discover the same underlying issue. Collapse near-duplicate comments before posting rather than making the developer reconcile them.

### 5. Suppress historically low-value categories

Use feedback to remove comment classes developers consistently reject. Uber reports stylistic/readability nits and minor logging/performance suggestions as lower-value than correctness and error-handling findings.

### 6. Instrument review quality as a product metric

Collect at least:

- explicit useful/not-useful feedback
- whether the author addressed the comment
- precision/recall/F1 on a curated benchmark
- latency and cost per review
- category/language/reviewer origin

Use a stable benchmark of real known bugs to compare model/prompt configurations rather than selecting models by reputation.

### 7. Keep deterministic checks deterministic

If a rule is cheap and reliably enforceable with a linter/static analyzer, keep it there. Use LLM review for semantic rules that normal analyzers cannot reliably identify.

## Repeated evaluation for stochastic reviewers

A single rerun is weak evidence that an issue disappeared because reviewer output is stochastic. Uber re-runs review multiple times on final code and uses semantic matching to estimate whether a posted issue was actually addressed.

The reusable principle is:

```text
stochastic detector -> repeated sampling or calibrated benchmark
```

rather than treating one model call as ground truth.

## Why it is useful

The central optimization is **precision over comment volume**. Review agents lose trust quickly when they are noisy; filtering and feedback loops are system-design requirements, not prompt polish.

## Caveats

- Production usefulness metrics are organization-specific; do not copy Uber's thresholds blindly.
- Review based only on source code will miss system-level facts such as feature flags, schemas, rollout state and historical decisions unless those contexts are explicitly connected.
- Multiple LLM stages increase latency and cost; compare gains against a single-reviewer baseline.
- Developer "addressed" behavior is useful telemetry but not proof that the original comment was correct.

## Prototype experiment

On 50-100 historical PRs with known review outcomes, compare:

1. single generic reviewer;
2. 3 specialized reviewers;
3. specialized reviewers + independent grader + dedupe.

Measure precision, recall, comments/PR, developer-relevant findings, cost, and latency. Prefer the smallest architecture that reaches an acceptable precision target.

## Sources

- https://www.uber.com/us/en/blog/ureview/

## Related

- [Evidence-gated coding-agent edits](evidence-gated-coding-agent-edits.md)
