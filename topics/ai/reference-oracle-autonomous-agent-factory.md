# Build autonomous coding-agent factories around an independent reference oracle

## What it is

When a system has a machine-readable specification **and an independently executable reference implementation**, split autonomous development into three bounded jobs: generate and validate reference tests, find divergences in the candidate, and repair one divergence at a time. Keep test evidence, issue lifecycle and merge policy outside any one coding agent. This is a production case study from AWS Emulator (November 11, 2026), **not** proof that arbitrary software can be built without human review.

The main reusable insight is **a continually growing, independently validated fidelity corpus**, rather than one model writing implementation and tests that agree with each other's errors. AWS's reference is the actual AWS API, while Smithy API models define request/response shape.

## Use when

- implementing an API-compatible replacement, emulator, parser, protocol gateway, legacy migration or other behavior with a callable reference;
- the external oracle can be queried under an explicit data/cost budget and its outputs can be retained legally and safely;
- many independent, narrowly scoped failures can be repaired in parallel;
- you need acceptance criteria stronger than “the agent's tests pass.”

Avoid this for tasks with subjective requirements, no safe reference, confidential data that may not enter test artifacts, or real-world effects that cannot be reset.

## System shape

~~~text
machine-readable spec + independent reference
        |
  TEST WRITERS [read specs, limited reference access; no candidate edits]
    -> proposed realistic scenario
    -> execute against reference in disposable account
    -> record normalized request, response, state effects, oracle/version
    -> only accept reproducible valid scenarios
        |
  immutable reference-validated fidelity corpus
        |
  ISSUE FINDERS [read-only candidate + corpus]
    -> replay candidate
    -> compare semantics, not unstable IDs/timestamps
    -> cluster likely common-cause failures
    -> durable issue per root cause, with reproducer
        |
  FIXERS [one issue per isolated checkout; no oracle mutation rights]
    -> source investigation -> smallest patch -> local tests
        |
  independent merge gate:
    compile / deterministic tests / reference fidelity regression
    security checks / reviewer / policy and branch protections
        | fail -> bounded rework / dead-letter issue
        | pass -> serialized merge to main
        |
  release branch -> cumulative retest -> artifact -> publish
~~~

### Roles and authority boundaries

- **Test writer:** may query the specification and a *disposable, rate-limited* reference. It cannot change the product code or the protected golden suite directly. Reference tests should exercise meaningful state transitions, not only static request/response shapes.
- **Issue finder:** replay the exact accepted suite against a pinned candidate commit, identify divergences, deduplicate common root causes and write evidence-backed issues. It should not invent requirements when a test is absent.
- **Fixer:** gets one issue, the pinned test evidence and a fresh worktree. It changes only allowlisted product paths. It cannot edit the reference tests, test evaluator, security policy or merge gate to make its fix pass.
- **Control plane:** owns stable issue IDs and states (NEW -> CLAIMED -> PATCH -> VERIFY -> DONE / REWORK / ESCALATED); handles expired claims, retries, concurrent changes and merge queue. Check actual PR/CI state after interruption rather than trusting a reported “done.”

Example issue contract:

~~~json
{
  "id": "fidelity/sqs/receive-then-delete",
  "base_sha": "<candidate-sha>",
  "oracle_version": "<service/model-version>",
  "test_ids": ["sqs.lifecycle.receive-delete.v2"],
  "repro": "replay --test sqs.lifecycle.receive-delete.v2",
  "expected": "reference-normalized-fixture.json",
  "actual": "candidate-normalized-output.json",
  "owner": null,
  "attempt": 0,
  "max_attempts": 3
}
~~~

Use a fresh reference snapshot/version whenever the remote contract materially changes; do not treat a real service's current output as an eternal constant. Include timestamps, SDK/model versions, locale/region and resource setup in each fixture identity.

## Evidence from AWS Emulator

AWS describes three concurrent agent teams: test writers generate AWS scenarios from Smithy models and first validate them against real AWS; issue finders replay tens of thousands of those tests against the emulator and cluster failures; fixers tackle one issue and pass build, coding review, security and fidelity gates. The internal repository is CodeCommit; after cumulative release-branch testing, source is exported to public GitHub, with open/fixed fidelity issues represented as directories.

The public Rust workspace has one crate per service, a service's Smithy model and operation implementations, a shared router, and directories for issues. This makes the work partitionable, but the public repo alone does not reproduce AWS's internal reference accounts, prompts, fixture store, worker scheduler or merge policy.

AWS reports hundreds of concurrent agents and thousands of fixed issues per day. These are first-party throughput claims, **not** independently verified end-to-end correctness, cost-effectiveness or safety metrics. A known failure in an earlier prototype was an agent building its own Python interpreter rather than reusing a library; explicit dependency-reuse policy was then added.

## Implementation cautions

- An external reference can be nondeterministic or stateful. Normalize only documented nondeterminism, isolate test accounts, reset resources, enforce rate limits, and store actual oracle outputs with versions. Never record live customer data in fixtures.
- Conformance ≠ completeness: the corpus covers observed scenarios, not the full product contract. Measure coverage by operation, state transition and negative case. Keep independently seeded mutants/hidden cases to catch a flawed or overfit corpus.
- If test writers and fixers share too much context, they can collude accidentally. Freeze test identities and reject unauthorized edits to oracle fixtures.
- When many agents merge rapidly, stale branches and flaky tests can erode fidelity. Run the suite against the **merged** exact SHA and periodically against a fresh reference.
- The vendor's architecture does not establish that “zero human code review” is appropriate for payments, identity or safety-critical systems. Preserve human approval for risk classes that need it.

## Prototype experiment

Build a small protocol emulator against a sandbox reference (e.g. a Redis/HTTP API subset). Define 20 stateful scenarios and 10 negative/error scenarios before implementation. Run separate writer/finder/fixer agents with scoped tools for 50 issues, using a merge queue with three retries maximum. Hold out 10 independently written scenarios and mutants.

Measure: reference validity, hidden-scenario pass rate, regression escapes after merge, issue duplication, retries, time and tokens per *accepted* fix, CI minutes and manual audit findings. Compare to a single implementer+test-writer agent. Stop if reference cost or safety makes the experiment impractical.

## Sources

- AWS, *How we built the agents that built AWS Emulator*, 2026-11-11: https://www.aboutamazon.com/news/aws/how-autonomous-ai-agents-built-aws-emulator
- Public source, issue directories and service layout: https://github.com/aws/aws-emulator
- AWS API model source: https://github.com/aws/aws-models

## Related

- [Deterministic outer loop](deterministic-outer-loop-agent-platform.md)
- [Event-reducer orchestration](event-reducer-coding-agent-orchestration.md)
- [Agent-generated test mutation gates](../testing/agent-generated-test-mutation-gates.md)
- [Evidence-gated coding-agent edits](evidence-gated-coding-agent-edits.md)
