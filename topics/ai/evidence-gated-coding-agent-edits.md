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

## Include external truth when code is not the source of truth

Repository evidence is sometimes insufficient. Before an agent simplifies or removes behavior that depends on deployed state, fetch the authoritative runtime/control-plane value first.

Examples:

- feature-flag cleanup -> current rollout percentage and target value;
- schema migration -> deployed schema/version;
- dependency removal -> runtime/config usage, not only imports;
- infrastructure cleanup -> actual resource/traffic state.

If that external state is ambiguous, insert a human checkpoint **before code mutation**, not after the patch has already committed to an interpretation.

A practical gate can look like:

```yaml
pre_edit:
  repo_references_discovered: true
  authoritative_runtime_state_fetched: true
  target_state_unambiguous: true
  target_state_confirmed: true   # human when ambiguity requires judgment
```

DoorDash's 2026 stale-feature-flag cleanup system uses exactly this shape: an analysis phase reads live rollout metadata before editing, produces a structured report, and asks an engineer to confirm the target value for ambiguous/partial rollouts.

## Isolate parallel edits and gate PR creation deterministically

When multiple coding agents work concurrently, give each task an isolated git worktree (or equivalent disposable workspace). Do not let parallel agents share a mutable checkout.

Then make PR creation conditional on machine-checkable gates, for example:

```yaml
submission_gate:
  build: passed
  tests: passed
  patch_coverage: ">= 95%"
  static_analysis: passed
  residual_references: 0
```

The exact thresholds are project-specific. The reusable rule is that the agent **cannot open/submit a PR just because it says it is done**; the harness verifies the required outputs first.

For Gradle builds running concurrently in multiple worktrees, consider `--no-daemon` when shared daemon state creates cross-worktree interference. DoorDash reports using isolated worktrees plus `--no-daemon`, hard per-agent timeouts, patch-coverage checks, tests, and Detekt before allowing its cleanup agents to open PRs.

## Why it is useful

FixedBench tested coding agents on 200 human-verified tasks where no source-code change was required. The evaluated agents still proposed undesirable source changes in 35–65% of cases. Explicitly asking agents to reproduce before patching reduced the problem, but also created over-abstention on partially fixed issues.

A newer evidence-conditioned execution experiment (ECLoop) goes further by enforcing structured, task-specific evidence before edit/submission actions. Across 500 SWE-bench Verified instances and multiple model/scaffold combinations, the paper reports Pass@1 improvements of 4.8–11.8 percentage points while also reducing average token use in the tested configurations.

DoorDash provides production evidence for extending the same principle beyond bug reproduction: its stale-feature-flag agent first grounds the intended edit in live rollout state, then performs each confirmed cleanup in an isolated worktree and blocks PR creation on build/tests/patch coverage/static analysis. In its reported sample of 50 recent flags, 45 produced usable PRs; the observed failure boundary was incomplete cleanup rather than silent target-value guessing. Treat those rates as workload-specific, not a general coding-agent benchmark.

## Caveats

- `Reproduce before patch` alone is too crude. A partially fixed issue can legitimately need a patch even if the original reproduction no longer fails exactly as described.
- A passing reproduction test does not prove the implementation is generally correct; still run relevant surrounding tests and inspect the behavioral contract.
- Evidence gates should be checkable from tool/runtime events when possible. Do not accept the model merely saying that it inspected something.
- External state must come from an authoritative source and be captured close enough to the edit that it has not gone stale.
- Worktree isolation prevents edit collisions; it does not protect shared external systems or shared build caches by itself.
- Excessive mandatory evidence can make trivial changes expensive. Gate only commitment-relevant evidence.

## Sources

- https://arxiv.org/abs/2605.07769 — *Coding Agents Don't Know When to Act* (2026-05-08)
- https://arxiv.org/abs/2607.28815 — *Preventing Premature Commitment in Coding Agents with an Evidence-Conditioned Execution Layer* (2026-07-30)
- https://careersatdoordash.com/blog/automating-feature-flag-cleanup-at-scale-with-a-multi-agent-llm-system/ — live-state grounding, human target confirmation, isolated worktrees and deterministic PR gates (2026-08-24)
