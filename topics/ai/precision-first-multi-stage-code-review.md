# Precision-first multi-stage AI code review

## What it is

A code-review agent architecture that treats review as a pipeline rather than one prompt: a cheap scout focuses attention, specialized issue generators investigate likely risks, separate verification tries to disprove findings, semantic deduplication collapses overlaps, and low-value categories are suppressed using production feedback.

Uber, DoorDash, Cloudflare and Wealthfront independently converge on this shape: spend expensive reasoning only where the diff or domain evidence suggests risk, and optimize for comments developers actually act on rather than comment volume.

## Use when

Use this when AI code review is producing too many false positives, duplicate comments, style nits, or low-value suggestions that developers ignore.

It is especially appropriate when review volume is high enough that a single generic reviewer becomes noisy and hard to tune.

## Pipeline

```text
changed files
  -> eligibility filter
  -> scout / risk localization
  -> focused evidence collection
       -> code paths / callers
       -> review-specific domain rules
       -> historical decisions / incidents
  -> specialized reviewers
       -> correctness
       -> best practices
       -> security
  -> disprove / independent grader
  -> confidence thresholds
  -> semantic dedupe
  -> category/value filter
  -> inline comments
  -> developer feedback + addressed/not-addressed telemetry
```

## Implementation rules

### 1. Filter low-signal inputs before the model

Exclude generated code, known experimental directories and other file classes where review comments are rarely useful.

### 2. Separate noticing from verifying

Use a cheap first pass to identify *where* deeper review is warranted instead of asking every reviewer to exhaustively analyze every line.

```text
scout(diff) -> [suspect region A, deleted contract B, unusual mapping C]
```

Then give deep reviewers the suspect region plus enough surrounding context to verify or reject the hunch.

DoorDash reports this split lets expensive reviewers spend attention on a handful of risk areas rather than diluting it across the whole diff.

### 3. Split review by concern

Use separate prompts/context for issue classes that need different evidence or thresholds. Examples:

- correctness / logic
- exception/error handling
- security
- organization-specific best practices

Do not assume one prompt is optimal for all categories.

### 4. Collect evidence with bounded research questions

Do not tell a subagent to "find relevant files" in a large repository; that objective is unbounded and tends to make everything look relevant.

Instead ask specific questions, for example:

```text
- Which callers rely on the removed null behavior?
- Where else is this enum mapped to a persisted value?
- What invariant governs writes to this table?
```

Wealthfront used cheap research agents for bounded questions, then **discarded their conclusions and kept the primary files/sources they found**. A stronger model received those sources and decided whether more research was needed. This limits subagent error propagation while retaining cheap parallel exploration.

Useful pattern:

```text
planner/reviewer
  -> bounded research question
  -> cheap search agent
  -> primary evidence + relevance scores
  -> reviewer judgement
```

### 5. Maintain review-specific domain profiles

Do not assume `AGENTS.md`/`CLAUDE.md` authoring instructions are ideal review context. Extract a smaller review profile containing invariants, architectural boundaries, known anti-patterns, incident lessons and recurring senior-review rules.

Potential sources:

- repository instructions, filtered to review-relevant rules;
- historical human review comments;
- ADR/design decisions;
- incident/postmortem lessons;
- operational rules not encoded in static analysis.

Keep provenance with each rule so a reviewer can fetch the underlying evidence when needed.

### 6. Try to disprove every candidate before posting

Before a finding reaches a developer, run an explicit falsification step:

```text
candidate finding
  -> search for counterevidence / safe caller / existing guard
  -> if disproved or weakly supported: drop
  -> otherwise: post with concrete evidence
```

This is stronger than asking the same model for a generic confidence score because the stage has an adversarial objective: prove the candidate wrong.

### 7. Grade every surviving candidate separately

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

### 8. Deduplicate semantically

Multiple reviewers can discover the same underlying issue. Collapse near-duplicate comments before posting rather than making the developer reconcile them.

### 9. Suppress historically low-value categories

Use feedback to remove comment classes developers consistently reject. Uber reports stylistic/readability nits and minor logging/performance suggestions as lower-value than correctness and error-handling findings.

### 10. Instrument review quality as a product metric

Collect at least:

- explicit useful/not-useful feedback;
- whether the author addressed the comment;
- precision/recall/F1 on a curated benchmark;
- acceptance rate by severity/category;
- latency and cost per review;
- category/language/reviewer origin;
- break-glass / bypass frequency for mandatory review gates.

Use a stable benchmark of real known bugs to compare model/prompt configurations rather than selecting models by reputation.

DoorDash reports 60.2% of settled high/critical findings caused a code change in its measured sample. Cloudflare reports 131,246 review runs over 48,095 merge requests in 30 days with deliberately low findings/review and only 0.6% break-glass use. These are evidence that precision-first review can work at scale, not transferable target thresholds.

### 11. Keep deterministic checks deterministic

If a rule is cheap and reliably enforceable with a linter/static analyzer, keep it there. Use LLM review for semantic rules that normal analyzers cannot reliably identify.

## Repeated evaluation for stochastic reviewers

A single rerun is weak evidence that an issue disappeared because reviewer output is stochastic. Uber re-runs review multiple times on final code and uses semantic matching to estimate whether a posted issue was actually addressed.

The reusable principle is:

```text
stochastic detector -> repeated sampling or calibrated benchmark
```

rather than treating one model call as ground truth.

## Why it is useful

The central optimization is **precision over comment volume**. Review agents lose trust quickly when they are noisy; attention routing, evidence collection, falsification and feedback loops are system-design requirements, not prompt polish.

## Caveats

- Production usefulness metrics are organization-specific; do not copy reported thresholds blindly.
- Review based only on source code will miss system-level facts such as feature flags, schemas, rollout state and historical decisions unless those contexts are explicitly connected.
- Mining Slack/review history can encode stale or contradictory tribal knowledge; retain provenance and freshness.
- Multiple LLM stages increase latency and cost; compare gains against a single-reviewer baseline.
- Developer "addressed" behavior is useful telemetry but not proof that the original comment was correct.

## Prototype experiment

On 50-100 historical PRs with known review outcomes, compare:

1. single generic reviewer;
2. scout + specialized reviewers;
3. scout + bounded evidence research + specialized reviewers + disprove pass + dedupe.

Measure precision, recall, comments/PR, acceptance, known-bug coverage, cost and latency. Prefer the smallest architecture that reaches an acceptable precision target.

## Sources

- https://www.uber.com/us/en/blog/ureview/
- https://careersatdoordash.com/blog/doordash-built-an-ai-code-reviewer-engineers-actually-listen-to/
- https://eng.wealthfront.com/2026/08/03/experiments-with-ai-code-review/
- https://blog.cloudflare.com/ai-code-review/

## Related

- [Evidence-gated coding-agent edits](evidence-gated-coding-agent-edits.md)
- [Engineer agent fleets with outcome-denominated unit economics](agent-fleet-unit-economics.md)
