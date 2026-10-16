# Gate coding-agent contributions on repository policy, including the trajectory

## What it is

A code patch can pass all functional tests and still be an unacceptable contribution: forbidden dependency additions, unapproved file changes, required test commands never run, misleading PR descriptions, broken contribution conventions, or unsafe intermediate actions. Turn the **applicable repository policies** into testable, versioned contracts and check two different things:

1. **Execution process:** what the agent *did* (tools invoked, paths touched, tests actually run, credentials/capabilities used, approval events).
2. **Final deliverable:** what it *left behind* (diff, tests, build metadata, commit, pull request, evidence).

SWE-CC is an open 2026 benchmark demonstrating the gap: it extracts 823 atomic policies from 12 real repositories and evaluates 500 end-to-end tasks using deterministic checkers. The generalizable design is **policy extraction -> explicit trigger -> runtime/final evidence -> deterministic verdict**, not a new prompt that says "please follow the rules."

## Use when

- running issue-to-PR or code-migration agents unattended or at scale;
- publishing from a repository with contribution, security, quality or disclosure requirements;
- comparing agent harnesses beyond SWE-bench-style test pass rates;
- ensuring prohibited intermediate actions remain visible even if the final diff is clean;
- moving recurring human review comments into reproducible CI checks.

## Policy contract

Compile only machine-checkable rules into blocking checks. Keep provenance and applicability:

```yaml
id: kb.intake.sources-logged
origin: CONTRIBUTING.md#source-log
scope: intake-pr
trigger:
  changed_paths: ["intakes/**", "topics/**"]
evidence: [base_sha, head_sha, changed_files, source_log_diff]
phase: final
checker: require_processed_source_log_for_intake
severity: merge-blocking
```

The `checker` identifier here is illustrative. A policy has four possible verdicts:

- `pass`: applicable and verified;
- `fail`: applicable, verified violation;
- `not_triggered`: change never entered the policy's scope;
- `withheld`: applicable but evidence is missing, incomplete, ambiguous or unparsable.

Do **not** quietly count `withheld` as a pass. Fail closed for critical rules and require escalation for other unknowns; give agents a structured error showing what evidence to supply.

## Implement the gate

### 1. Discover and atomize rules

Read `CONTRIBUTING.md`, `AGENTS.md`, nested instructions, security rules, code-owner conventions, test requirements and existing CI. Split into atomic requirements. Record the exact source revision, affected paths/task classes and intended authority.

For each candidate, decide:

```text
precise + deterministic consequence -> CI / tool-level enforcement
context-dependent but reviewable    -> advisory / human owner
ambiguous aspirational guideline    -> retain in documentation, do not hard-gate
```

Use unit tests for each checker: one pass, one fail, one not-triggered, and a withheld case. A validator with frequent false positives erodes trust.

### 2. Capture execution outside the agent

Use a tool gateway or harness wrapper to record *authoritative* invocations and outcomes: command arguments (redacted), exit status, file writes, approvals, checkout SHA, test invocation and timestamps. Never trust the agent's end-of-run narrative as proof that tests executed. Avoid leaking credentials or sensitive command output into the audit trail.

```text
repo docs + task + pinned base SHA
       -> applicable policy set
       -> agent in isolated workspace
       -> externally recorded tool/command events
       -> candidate commit and PR
       -> replay/evaluate runtime rules
       -> evaluate final diff/metadata rules
       -> CI status bound to exact head SHA
       -> human/merge gate
```

Examples of **runtime** rules: "run targeted tests after the last code-changing edit", "never mutate protected directories", "request approval before applying a database migration." Examples of **final** rules: "new API endpoint has an integration test", "source-log entry exists", "PR contains required evidence fields".

### 3. Bind every check to a particular state

Use repository, base SHA, head SHA, policy version, event-log ID and checker version as evaluation identity. A new commit invalidates checks about the old head. A policy change may require re-evaluation even if the patch is unchanged. For rules about *past behavior*, a clean final diff cannot erase an earlier violation.

### 4. Report both coverage and compliance

```text
triggering_rate = (pass + fail + withheld) / total_policies
compliance_rate = pass / (pass + fail)
```

Always publish both and the `withheld` count. A do-nothing agent can trigger few policies and appear artificially compliant. Also track functional success, severity of policy failures, human correction minutes, runtime cost and false-positive disputes separately.

## Try the open SWE-CC implementation

SWE-CC supports Python 3.11 and includes example trajectories, frozen policy corpora, checker unit tests and adapters for two agents. After cloning the upstream repository:

```bash
python3.11 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt -e .
pytest -q
scripts/quickstart.sh

# Score a stored run and inspect if the model actually retrieved policies:
swe-cc check <trajectory.json> --project sympy --setting native --jsonl rows.jsonl
swe-cc retrieval <trajectory.json> --project sympy
```

For your own agent, adapt its externally collected events to SWE-CC's JSON trajectory protocol; see upstream `docs/evaluate-your-agent.md` and `harness/probe.sh` / `harness/collect.sh`. To enforce policies on **your own repository**, author its own corpus and checkers (the published corpus targets twelve specific projects).

## Evidence and caveats

The October 2026 SWE-CC authors report that the evaluated agents violated 43.1% of applicable project policies despite many patches resolving the functional task. Consolidating policies into model context increased average compliance relative to making the model retrieve them, but the best tested mean remained 67.0%. Nearly half the violations occurred during execution rather than only in the final deliverable. Those figures come from the authors' task/repository/model distribution; they are not a predicted rate for our codebase.

The released evaluation software contains deterministic checkers and summarized outcomes from 8,000 benchmark runs, **not all 8,000 raw trajectories**, so not every historical judgment can be independently replayed from the public release. Policy applicability and judge correctness still need spot audits. Some important style/architecture requirements remain non-deterministic and should stay review items rather than brittle CI blockers.

Don't let a policy checker become a privilege escalation path: run its code separately from agent-controlled files, pin its version and constrain any evaluator commands.

## Prototype experiment

In `Ra13s/technical-knowledgebase`, extract 8–12 meaningful policies (source log required, intake begins with synthesis, references resolve, only intended paths change, no stale source duplicated, current-head CI evidence, no direct push to main). Encode deterministic subset as an always-running PR check. Store tool events from 10 agent runs and check compliance at both intermediate and final phases. Compare with functional/Markdown validation alone. Reject gates that cannot produce reliable fail/not-triggered/withheld evidence.

## Sources

- Truong et al., *Correct Code, Broken Contributions? SWE-CC* (2026-10): https://arxiv.org/abs/2610.06193
- SWE-CC implementation and runner: https://github.com/dangtruong01/swe-cc-arxiv

## Related

- [Evidence-gated coding-agent edits](evidence-gated-coding-agent-edits.md)
- [Gate agent harness configuration as a supply chain](agent-harness-supply-chain-gates.md)
- [Deterministic outer loop for coding-agent platforms](deterministic-outer-loop-agent-platform.md)
- [Evaluate AI code reviewers](reviewbench-code-review-evaluation.md)
