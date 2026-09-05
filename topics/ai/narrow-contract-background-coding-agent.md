# Narrow-contract background coding agent

## What it is

A background coding-agent operating model for bounded work: only start from an explicit trigger, reject underspecified tasks before coding, produce a small reviewable PR, and keep human approval and existing CI/security gates intact.

Checkout.com reports using this shape for Agent HAL, which generated 18% of its PRs at the time of publication.

## Use when

Use this for recurring or queued tasks that are useful but low-context and well scoped, for example:

- dependency/security advisory remediation
- small internal-product changes
- bounded endpoints or UI changes
- repetitive maintenance across many repositories
- tickets that can be expressed with explicit acceptance criteria

Do not use it as an ambient agent that invents its own work.

## Workflow

```text
explicit trigger
  -> readiness gate
  -> isolated implementation run
  -> fresh-context spec reviewer
  -> CI/security gates
  -> PR lifecycle maintenance
  -> named human approval
  -> merge
```

### 1. Explicit triggers only

A run must trace to a labelled ticket, scheduled job, or activity on a PR already owned by the agent. Avoid open-ended "find something useful to do" autonomy.

### 2. Gate on task readiness before editing

Score/check the ticket for concrete scope and acceptance criteria. If it is too thin, stop and request clarification instead of guessing requirements.

A minimal machine-readable gate could require:

```yaml
agent_ready:
  problem: required
  target_repo: required
  acceptance_criteria: required
  allowed_scope: required
  validation: required
```

### 3. Separate implementation and review contexts

Before opening the PR, run a second reviewer persona in a fresh context against the original ticket and the produced diff. Do not let the same conversational context both interpret the spec and certify that interpretation.

```text
implementer context != spec-review context
```

If the fresh reviewer finds missing acceptance criteria, return the task to implementation or stop.

### 4. Keep the PR healthy while it waits

The background workflow should own mechanical branch maintenance: rerun/fix failing CI where permitted and rebase/update when `main` moves. Human review should begin on a current, mergeable change rather than a stale branch.

### 5. Use the same gates as human PRs

Do not create an AI fast lane. Run the normal tests, security scans, branch rules and named approvals. Checkout.com explicitly keeps HAL from auto-merging its own code.

## Why it is useful

The key optimization is cognitive offload, not maximum autonomy. Narrow contracts make diffs small enough to review properly and allow background agents to absorb interruption-heavy work without creating a second, AI-specific SDLC.

## Failure modes / caveats

- **Vague ticket -> confident wrong implementation.** Reject before coding.
- **Self-review in the same context -> correlated interpretation error.** Use a fresh-context reviewer.
- **Automation complacency -> rubber-stamped PRs.** Keep named human approval and small diffs.
- **Stale PRs -> review burden returns.** Let the workflow maintain branch/CI health.
- A vendor case study shows feasibility, not that 18% PR share is a target for other teams.

## Prototype experiment

Run this on 20 low-risk backlog tickets. Compare against normal agent-assisted implementation using:

- acceptance/rejection at readiness gate
- first-pass PR acceptance rate
- human review minutes
- CI repair iterations
- reopened/reverted changes
- time from ticket-ready to review-ready PR

## Sources

- https://www.checkout.com/blog/how-agent-hal-ships-software-checkout-com

## Related

- [Evidence-gated coding-agent edits](evidence-gated-coding-agent-edits.md)
- [Bind agent approvals to the exact side effect](enforcement-bound-agent-approvals.md)
