# Gate agent-generated tests on validity and defect-detection strength

## What it is

Judge a generated test suite against **two independent oracles**:

1. **Validity:** do the tests pass against a trusted reference implementation, including across repeated runs?
2. **Strength:** do they fail against deliberately faulty implementations of specified behavior?

Do not treat successful execution, high statement coverage, number of assertions, or a passing CI job as proof that newly written tests can detect bugs. The October 2026 **TestSuiteBench** study evaluates 165 specification-to-test tasks with an average of 74 requirement-targeted mutants each. Its strongest tested agent killed only about 55% of mutants even though most generated tests were valid. It is evidence for separate test-quality gates, not a universal score target.

## Use when

- a coding agent writes tests for a new module, API contract or bug fix;
- a planner/reviewer wants independent evidence that generated tests meaningfully constrain behavior;
- setting up autonomous implementation/validation loops for services with few existing tests;
- evaluating test-writing strategies or test-agent prompts on a frozen corpus.

## Workflow and contract

~~~text
versioned specification + immutable reference artifact
    -> agent generates tests in isolated workspace
    -> run against reference (validity, compile/run determinism)
    -> evaluate against independently created faulty variants
       -> requirement-targeted mutants preferred
       -> generic bytecode mutants useful supplementary signal
    -> collect uncovered requirements and surviving mutants
    -> independent reviewer checks assertions and test intent
    -> accept / request better tests / escalate ambiguous behavior
~~~

**Avoid circular validation:** A test written by the same agent that implemented the feature and run only against that agent's implementation can pass while both share the same mistaken assumption. Fix the reference specification and defect set outside the candidate agent's editable workspace.

Record the evaluation identity as `{spec_revision, reference_sha, mutant_set_version, test_sha, runner_version}`. Preserve a private holdout for prompt/harness comparisons.

### TestSuiteBench reproducible experiment

The public repo has 165 CLI applications (mostly compiled Rust/Go/C++) and uses Linux x86-64 Docker images. The agent sees a detailed specification but not the correct executable or mutants at generation time. It produces `tests.tar.gz` with an `eval/run.sh` that tests `./executable` and writes JUnit XML to `eval/results.xml`.

~~~bash
git clone https://github.com/eth-sri/testsuitebench.git
cd testsuitebench
uv sync
# First choose a small task subset; use official docs for model-run identifiers.
bash scripts/eval/run_agents.sh --runs "codex-gpt-5.6-sol" \
  --tasks-file ground_truth/valid_task.txt --jobs 2 -- --part test_gen
uv run programbench eval-tests \
  output/eval/testgen/<model>_<reasoning> \
  --workers 4 --mutant-workers 8
~~~

Requires a supported agent login/API credential. Fetches ground-truth binaries, reference tests and mutants on demand from HuggingFace. A complete offline cache is roughly 100 GB. **Never give the candidate agent access to the mutant implementations or reference tests in the actual held-out task.**

### Pragmatic Java/Spring pilot

On an existing Maven/JUnit service, PIT provides inexpensive bytecode mutations for the touched packages:

~~~bash
# Pin and configure pitest-maven (including the JUnit 5 plugin if required).
# Limit targetClasses to changed production packages and targetTests to relevant tests.
mvn test-compile org.pitest:pitest-maven:mutationCoverage
# Inspect target/pit-reports/... and surviving mutations.
~~~

PIT's standard mutations (conditional negation, return-value changes, etc.) are **not equivalent** to TestSuiteBench's curated, requirement-specific mutants. Use PIT as an initial automated signal. For important contracts, seed 5–10 hand-reviewed defects, e.g. allow an unauthorized actor, omit a transaction rollback, ignore one tenant ID, or change an API boundary value. The evaluator must prove these seeded defects are detectable by a known-good test suite; discard equivalent/unobservable mutants. Do not blindly set one global mutation-percentage release gate before calibrating against the service and its critical behaviors.

Report separately:

| Signal | What it means |
|---|---|
| Reference pass rate | Candidate tests are compatible with the intended behavior |
| Repeated-run stability | They are not dependent on chance or live external conditions |
| Relevant mutant kill rate | They catch the selected incorrect behaviors |
| Uncovered requirement count | There are specification regions with no meaningful test |
| Escaped defect / human repair | The gate improves actual engineering outcomes |

## Failure modes

- A mutant may be behaviorally equivalent or unrealistic; failing to kill it is not necessarily a test defect.
- Tests might only assert superficial output, or accidentally depend on implementation details.
- Low mutation scores on a broad package may reflect irrelevant generated/defensive code; segment by critical behavior.
- A passing generated test may exercise an agent's mistaken shared assumption. Independent specifications, regression fixtures and adversarial mutants reduce this risk.
- TestSuiteBench is a preprint on selected CLI programs; its measured scores do not establish Java/Spring production efficacy.
- Running all mutants in every PR is expensive. Use changed-package sampling in CI and deeper scheduled evaluation when appropriate.

## Experiment

Take 15 previously fixed real defects in one Spring service. Write independent requirements and a reference test baseline, then ask an agent for tests without showing the historical fixes. Compare (A) agent's self-tests, (B) independent agent-generated tests and (C) tests plus a mutation/reviewer gate. Count known bugs reintroduced and detected, valid tests, reviewer minutes, runtime and added test maintenance cost.

## Sources

- https://arxiv.org/abs/2610.13065
- https://github.com/eth-sri/testsuitebench
- https://pitest.org/quickstart/maven/

## Related

- [ReviewBench code-review evaluation](../ai/reviewbench-code-review-evaluation.md)
- [Repository policy compliance gates](../ai/repository-policy-compliance-gates.md)
- [Microcks provider contract tests](microcks-testcontainers-contract-tests.md)
