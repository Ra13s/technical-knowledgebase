# Evidence-gated coding-agent edits

## What it is

Before a coding agent edits production code or submits a patch, require observable repository evidence that the change is needed and that the proposed change has been investigated sufficiently.

This prevents a common agent failure mode: **acting because a task asks for a fix even when the reported issue is stale, already fixed, only partially understood, or contradicted by repository evidence**.

## Use when

Apply this to issue-driven autonomous or semi-autonomous coding agents, especially for:

- bug reports
- maintenance tickets
- flaky or stale issues
- unfamiliar repositories
- automated issue-to-PR pipelines

## Minimal agent instruction

Put an instruction like this in the coding-agent harness or repository instructions:

```text
Before editing production code for a bug report:

1. Establish concrete evidence that the reported problem still exists.
   Prefer a reproduction command/test or an observed failing behavior.
2. Inspect the code you plan to change and the relevant tests/callers.
3. If the issue is already fixed or cannot be reproduced, do not make a
   speculative code change. Report "no code change required" and include
   the evidence. This is a successful outcome.
4. If the issue is only partially fixed, identify and reproduce the
   remaining failing behavior before editing.
5. After the patch, run the reproduction/target test and the relevant
   surrounding test suite before declaring success.
```

The important part is step 3: **inaction must be an explicitly valid success path**. Otherwise agents tend to interpret the existence of a ticket as evidence that a code change is required.

## Stronger harness implementation

For higher-autonomy agents, do not rely only on prose instructions. Track evidence as structured runtime state and gate commitment actions.

Example evidence state:

```yaml
evidence:
  issue_reproduced: true
  target_code_inspected: true
  relevant_tests_inspected: true
  relevant_callers_inspected: true
  target_test_after_patch: passing
  surrounding_tests_after_patch: passing
```

Before allowing an edit or final PR submission, check the conditions relevant to that action. If a required condition is missing, return the unsatisfied evidence gap to the agent instead of executing the write/submit action.

Do not require every condition for every task. Compile the gate from the issue and repository structure; for example, a configuration-only change may have different evidence requirements from a business-logic bug.

## Why it is useful

FixedBench tested coding agents on 200 human-verified tasks where no source-code change was required. The evaluated agents still proposed undesirable source changes in 35–65% of cases. Explicitly asking agents to reproduce before patching reduced the problem, but also created over-abstention on partially fixed issues.

A newer evidence-conditioned execution experiment (ECLoop) goes further by enforcing structured, task-specific evidence before edit/submission actions. Across 500 SWE-bench Verified instances and multiple model/scaffold combinations, the paper reports Pass@1 improvements of 4.8–11.8 percentage points while also reducing average token use in the tested configurations.

## Caveats

- `Reproduce before patch` alone is too crude. A partially fixed issue can legitimately need a patch even if the original reproduction no longer fails exactly as described.
- A passing reproduction test does not prove the implementation is generally correct; still run relevant surrounding tests and inspect the behavioral contract.
- Evidence gates should be checkable from tool/runtime events when possible. Do not accept the model merely saying that it inspected something.
- Excessive mandatory evidence can make trivial changes expensive. Gate only commitment-relevant evidence.

## Sources

- https://arxiv.org/abs/2605.07769 — *Coding Agents Don't Know When to Act* (2026-05-08)
- https://arxiv.org/abs/2607.28815 — *Preventing Premature Commitment in Coding Agents with an Evidence-Conditioned Execution Layer* (2026-07-30)
