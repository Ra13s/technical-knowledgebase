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
  head_sha: 92b4f3a...
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

## Bind evidence to the exact code state and verify closure

Evidence expires when the code it describes changes. Treat test, review and CI results as assertions about an exact commit, not about a branch name or PR in general.

A gate should carry the repository identity it was produced against:

```yaml
verification:
  head_sha: 92b4f3a...
  tests:
    sha: 92b4f3a...
    status: passed
  static_analysis:
    sha: 92b4f3a...
    status: passed
```

If a rebase, force-push or repair commit changes the head SHA, invalidate prior results that depend on code content and rerun the required gates. Do not accept late CI notifications for the old head merely because they belong to the same PR.

Shopify's River vulnerability-remediation workflow applies this in production: after rebasing dependency fixes it discards CI results from the pre-push heads and waits for checks on the new commit.

Also distinguish **patch accepted** from **problem closed**. After merge, verify the authoritative postcondition on the default branch and external tracker/control plane.

For security remediation, for example:

```text
PR green       != vulnerability fixed
PR merged      != vulnerability fixed

default branch no longer vulnerable
+ dependency/runtime state confirms fix
+ vulnerability tracker reconciled
= closed
```

Valid terminal outcomes can include `fixed`, `already-fixed/superseded`, `rejected-with-evidence`, or `escalated-with-evidence`. Optimize for evidence-supported closure rather than PR count.

### Reconcile ledgers against live state

Agent ledgers, memories and task databases are useful coordination state, but they are claims rather than operational truth. At the start of a resumed run:

1. fetch current repository/PR/tracker state;
2. reconcile the saved ledger against it;
3. verify the issue still exists on the current head;
4. search for superseding fixes already in flight;
5. only then continue editing.

Put guarantees such as exhaustive pagination, deduplication, current-head identity and accounting in deterministic code. Prompts are appropriate for judgement; they are not reliable control-plane invariants.

### Hand off the smallest unresolved decision

When evidence reaches a boundary the agent cannot legitimately decide, preserve the current head, actions tried and evidence gathered, then ask the responsible owner the smallest question needed to resume.

If the desired outcome or evidence for correctness cannot be stated, another autonomous edit is a bet rather than a fix.

## Why it is useful

FixedBench tested coding agents on 200 human-verified tasks where no source-code change was required. The evaluated agents still proposed undesirable source changes in 35–65% of cases. Explicitly asking agents to reproduce before patching reduced the problem, but also created over-abstention on partially fixed issues.

A newer evidence-conditioned execution experiment (ECLoop) goes further by enforcing structured, task-specific evidence before edit/submission actions. Across 500 SWE-bench Verified instances and multiple model/scaffold combinations, the paper reports Pass@1 improvements of 4.8–11.8 percentage points while also reducing average token use in the tested configurations.

DoorDash provides production evidence for extending the same principle beyond bug reproduction: its stale-feature-flag agent first grounds the intended edit in live rollout state, then performs each confirmed cleanup in an isolated worktree and blocks PR creation on build/tests/patch coverage/static analysis. In its reported sample of 50 recent flags, 45 produced usable PRs; the observed failure boundary was incomplete cleanup rather than silent target-value guessing. Treat those rates as workload-specific, not a general coding-agent benchmark.

Shopify's River extends the pattern across the full remediation lifecycle. It revalidates stored claims against live repository/PR/vulnerability state, binds CI to the current head, and verifies the default branch plus dependency/tracker state after merge. Shopify reports that in the first 11 days of its dependency workflow the open backlog fell about 70%, with roughly two-thirds of handled items directly merged and the rest found obsolete/already fixed; freshness-gated security merges rose from about 10% to 80%. Those are Shopify-specific operating results, not general expected rates.

## Caveats

- `Reproduce before patch` alone is too crude. A partially fixed issue can legitimately need a patch even if the original reproduction no longer fails exactly as described.
- A passing reproduction test does not prove the implementation is generally correct; still run relevant surrounding tests and inspect the behavioral contract.
- Evidence gates should be checkable from tool/runtime events when possible. Do not accept the model merely saying that it inspected something.
- Evidence must be bound to the state it describes; after a head change, stale CI/test success is not transferable.
- External state must come from an authoritative source and be captured close enough to the edit that it has not gone stale.
- Worktree isolation prevents edit collisions; it does not protect shared external systems or shared build caches by itself.
- A ledger can become stale; reconcile it with live state before using it to drive side effects.
- Excessive mandatory evidence can make trivial changes expensive. Gate only commitment-relevant evidence.

## Sources

- https://arxiv.org/abs/2605.07769 — *Coding Agents Don't Know When to Act* (2026-05-08)
- https://arxiv.org/abs/2607.28815 — *Preventing Premature Commitment in Coding Agents with an Evidence-Conditioned Execution Layer* (2026-07-30)
- https://careersatdoordash.com/blog/automating-feature-flag-cleanup-at-scale-with-a-multi-agent-llm-system/ — live-state grounding, human target confirmation, isolated worktrees and deterministic PR gates (2026-08-24)
- https://shopify.engineering/river-vulnerability-remediation — current-head evidence, stale-CI invalidation, deliberate handoff and post-merge closure verification (2026-09-02)
