# Protect executable goal-level acceptance contracts from coding agents

## What it is

Separate the business acceptance contract from the code that implements it and the code that runs the checks. Store human-reviewable input, expected outcome and required evidence as data; let an independent verifier execute it against one immutable candidate artifact. Keep graded progress metrics separate from the binary release decision.

Use this when agents can edit both implementation and tests, when multiple UI or API steps must work together, or when refactoring must preserve established behavior. It complements mutation testing: the former protects what must be proven, the latter tests whether the proof detects defects.

## Implementation

Example: a campaign UI must resume the right scene and a completed campaign must stay completed. An independently maintained contract could look like this (illustrative YAML, not a third-party API):

~~~yaml
version: 1
core:
  - id: resume
    setup: {campaign: started, currentScene: 3}
    steps:
      - {action: openCampaign}
      - {action: clickContinue}
    mandatory:
      - {id: correct-scene, expect: {scene: 3}}
      - {id: no-first-scene-label, expect: {firstSceneLabel: false}}
  - id: completed
    setup: {campaign: completed}
    steps:
      - {action: openCampaign}
    mandatory:
      - {id: no-resume, expect: {continueVisible: false}}
      - {id: completion-preserved, expect: {status: completed}}
~~~

Use a strict schema, unique stable criterion IDs, nonempty core and mandatory lists, and a small allowlisted vocabulary. No executable expressions in fixtures. The test runner sets up isolated state, invokes the real public entry point (e.g. Playwright browser or HTTP API), observes response/UI and authoritative persisted state, and emits records with IDs, pass/fail/error, evidence references and artifact SHA. Seed data through a separate test-only adapter, not a production backdoor. Use JUnit parameterized tests for simpler CSV/JSON business rules rather than inventing a DSL unnecessarily.

### Separate roles and write boundaries

1. Product/developer contract owner approves core goals, scenarios and required checks before implementation; the agent may draft additions for separate approval.
2. Implementer can edit application code and developer tests, but cannot change the approved contract snapshot, evaluator, fixture seeding logic or CI acceptance logic.
3. Independent evaluator checks out those trusted components from the approved base revision (or a separately protected repository), builds candidate SHA once and runs it in a disposable, unprivileged environment. Test the packaged artifact or running service, not a second build per scenario.
4. Publisher requires a verified record for each required criterion. Missing, duplicate, unexpected or malformed records, timeout, incorrect artifact identity, or evaluator failure block acceptance. Do not infer success from the model's final message.
5. Contract changes need independent review; evaluate old unaffected criteria and the approved new criteria. Pin the approved contract revision rather than comparing blindly with a moving main branch.

~~~text
assert contract.core is nonempty
assert each core goal has nonempty mandatory criteria
assert record IDs exactly match the contract's expected IDs
assert every record binds to the same candidate artifact and run
assert all required records have verified evidence
accepted = all(required criterion status == PASS)
progress = passed / evaluated   # diagnostic only, never substitutes for acceptance
~~~

In CI protect the contract and evaluator through path restrictions/review rules plus a trusted check outside the candidate's write authority. CODEOWNERS alone is not a sandbox. Do not run untrusted candidate code in a privileged workflow with credentials or writable evaluator/contract files. Use isolated application runtime, restricted network, read-only mounted verifier data, and separate publication credentials. Preserve logs/screenshots under restricted retention; use deterministic assertions where possible and human/independent evaluation for subjective criteria. Do not accept an LLM's claim without evidence.

## Evidence and limits

JoyFrontBench (11 Nov 2026, preprint) uses goal-level criteria over 100 frontend problems. On its 70-problem test set, the five models with OpenHands averaged 87.69% task score but 51.52% whole-application success. This is a benchmark distinction, not a measured production failure rate. Matteo Vaccari's 11 Nov practitioner article proposes readable black-box fixtures and protection against unintended rewrites; its interview evidence is qualitative, not a controlled productivity study. An Agenor maintainer's separate case shows passing class tests did not exercise two annotation-driven behaviors through the real public entry point.

Tradeoffs: fixture DSLs need maintainers; some performance, security, accessibility and visual-quality requirements need specialized verification. Freeze time, seeds and external responses for replayable tests but retain realistic integration coverage. Never interpret a valid fixture as a complete specification. Split mandatory vs optional criteria deliberately; unnecessary mandatory checks inflate flakiness. For nondeterministic LLM behavior, use repeated scenarios and a predeclared acceptance rule rather than exact prose matches.

## Experiment

Choose 10 existing UI/API journeys and 5 known failure variants (wrong resume, skipped required step, state not persisted, stale cache, misleading success). Compare current CI plus agent self-tests with the protected contract runner. Measure independently seeded defects detected, whole-journey success, partial score, flaky retries, review minutes and fixture maintenance. Include an attempt to remove a required criterion or forge a success record; the gate must fail. Start in shadow mode before requiring the gate.

## Sources

- JoyFrontBench: https://arxiv.org/abs/2611.14952
- Matteo Vaccari, Tests as data: https://martinfowler.com/articles/tests-as-data.html
- Mauro Mura, The tests passed. The feature had never worked: https://foojay.io/today/the-tests-passed-the-feature-had-never-worked/

## Related

- [Mutation gates for agent-generated tests](agent-generated-test-mutation-gates.md)
- [Agent-addressable application feedback loops](../ai/agent-addressable-application-feedback-loop.md)
- [Reference-oracle agent factory](../ai/reference-oracle-autonomous-agent-factory.md)
- [Agentic CI taint-flow gates](../ai/agentic-ci-taint-flow-gates.md)
