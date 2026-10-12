# Evaluate AI code reviewers with grounded and augmented findings

## What it is

Use a reproducible, pull-request-level evaluation harness rather than measuring code-review agents by how many comments they post. GitHub's October 2026 **ReviewBench** provides a concrete, open implementation: 219 public pull requests from 187 repositories across 19 languages; mirrored base/head revisions; human-audited reference findings; a reviewer-container contract; and a judging pipeline.

Keep two questions distinct: **Did the reviewer find known valid defects?** (grounded precision/recall) and **Did it discover valid defects missing from the current reference set?** (augmented precision/recall). A fixed gold set is necessarily incomplete. But augmented recall's denominator varies with an agent's novel discoveries, so use **grounded recall** for between-system comparisons and augmented measures as supplemental, per-system diagnostics.

## Use when

- comparing reviewer models, prompts, scout/specialist configurations, independent verification stages or ensembles;
- deciding whether additional reviewer stages justify their token/latency cost;
- regression-testing code-review model or harness upgrades;
- testing whether high precision is merely hiding low recall;
- tuning severity/category-specific false-positive and missed-defect tradeoffs.

## Workflow

1. **Freeze the task corpus.** Pin the exact repository and base/head SHAs (and preferably mirrors) so upstream history edits do not change the reviewed diff. Keep a private holdout separate from tuning data.
2. **Build reference findings from independent sources.** Combine historical human reviews, follow-up fix commits, static analysis and different model families; semantically deduplicate the *underlying defect*, not just the comment text.
3. **Judge independently.** Apply one written rubric for truth, relevance and non-triviality. Have humans audit samples and version the golden findings, matcher, judge model/prompt and rubric.
4. **Run each reviewer in a fresh, isolated task context.** Give a read-only checkout and a bounded step/time/token budget. Normalize output into specific file/line, claim, evidence, severity and category fields.
5. **Score grounded and augmented findings separately.** For the grounded view, match candidate findings to validated known issues. For unmatched findings, use a separate validity judgement and sample human review. Report precision, recall/F1, missed high-severity findings, cost, p95 latency, comment volume and failed-task rate.
6. **Cross-check against production.** Shadow/A-B against live PRs with addressed findings, developer feedback, follow-up fixes and human reviewer workload. A benchmark is an offline promotion signal, not a replacement for production validation.

### Public ReviewBench smoke test

ReviewBench supplies a thin reviewer container contract, an example Codex CLI adapter and a script that tests the *adapter* on a frozen corpus:

~~~sh
git clone https://github.com/review-bench/ReviewBench
cd ReviewBench
# Build your own reviewer image using AGENT_CONTRACT.md and examples/codex-cli.
scripts/try-agent.sh my-reviewer:dev --pr 0 -e MY_API_KEY
scripts/try-agent.sh my-reviewer:dev -e MY_API_KEY
# The test set is 25 PRs; use --set full for the 219-PR corpus.
~~~

**Important:** `scripts/try-agent.sh` validates container execution and output shape; it does **not** judge the quality of the resulting findings. For private scoring, use the documented `npm run judge --` flow with your own model/judge credentials. Official submissions run the full 219-PR set three times; model keys remain the submitter's responsibility. Do not put secrets inside images or PRs.

### Internal evaluation shape

~~~text
PR (repo, base SHA, head SHA)
  -> frozen read-only checkout
  -> reviewer config A/B/C
  -> normalized findings JSON
  -> match known validated defects
  -> independent judgement of unmatched findings
  -> grounded + augmented metrics by severity/category
  -> human audit of disagreements
  -> promote / rollback decision
~~~

Add an organization-specific holdout of 30–100 Java/Kotlin/Spring PRs with real defects (transactions, auth/session boundary, race conditions, API contract changes and failure handling). An open-source benchmark alone cannot test whether the reviewer knows our architectural invariants.

## Evidence and limits

GitHub reports that senior engineers independently re-labeled golden findings with 96.6% true/false agreement. In one **GitHub-reported** multi-model ensemble production experiment, addressed rate improved 8.0%, estimated recall improved 13.6%, comment count rose 61%, and review cost fell 8.0%. These are vendor results from its own review product, not proof that other agent systems will obtain the same gains.

- The public golden set is incomplete; a judge can be confidently wrong. Retain examples and an appeals/human-audit path.
- **Augmented recall is not a direct cross-agent ranking metric** because the added reference denominator varies between agents.
- Public benchmarks are susceptible to memorization, prompt tuning and leakage. Prefer frozen local and private holdouts before deployment decisions.
- Developer-addressed rate is a proxy for usefulness, not proof that the finding was originally correct.
- Measure escaped defects and review burden, not just precision; a reviewer that says almost nothing can look superficially precise.

## Prototype experiment

Use 50–100 historical internal PRs with known outcomes. Compare: (A) current single reviewer, (B) scout plus verifier, and (C) ensemble or targeted specialists. Run repeated trials, audit unmatched findings, and graph severity-weighted grounded recall, augmented valid findings, noise, human-review minutes, latency and cost per useful comment. Prefer the smallest configuration meeting an explicit precision/recall target.

## Sources

- https://github.blog/ai-and-ml/reviewbench-an-open-benchmark-for-ai-code-review/
- https://github.com/review-bench/ReviewBench
- https://github.com/review-bench/ReviewBench/blob/main/AGENT_CONTRACT.md

## Related

- [Precision-first multi-stage AI code review](precision-first-multi-stage-code-review.md)
- [Gate agentic security findings with proof](agentic-security-presubmit-pipeline.md)
- [Agent fleet unit economics](agent-fleet-unit-economics.md)
