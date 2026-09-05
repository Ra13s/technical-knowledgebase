# Deterministic outer loop for coding-agent platforms

## What it is

A coding-agent platform pattern where the agent owns reasoning and edits inside an isolated session, while the surrounding workflow deterministically controls checkout state, validation, retries, publication and long-running orchestration.

Dropbox's Nova platform uses this shape across interactive sessions, CI repair, flaky-test remediation, migrations and dependency upgrades.

## Use when

Use this when coding agents need to run as reusable infrastructure rather than one-off chat sessions, especially when:

- tasks run asynchronously or in parallel
- the repository depends on organization-specific build/test infrastructure
- validation may take a long time
- failures need bounded retry/re-entry
- multiple models/agents should sit behind one execution interface

## Workflow shape

```text
caller
  -> fixed repo commit + task + validation plan
  -> isolated agent session
  -> candidate change
  -> deterministic validation outside the agent
       -> pass: publish/review
       -> fail: feed evidence back into session
  -> bounded retry count
```

Example request shape inspired by Nova:

```json
{
  "repo_commit": "<sha>",
  "task": "Investigate this CI failure and propose a fix",
  "validation_commands": [
    "bazel test //path/to:test_target",
    "bazel test //path/to/related:all"
  ],
  "continue_on_validation_failure": true,
  "max_iterations": 5,
  "push_branch": "ai/ci-fix"
}
```

## Key rules

### Snapshot every run

Start from an explicit commit, not a mutable working directory. This makes each agent run reproducible and allows parallel execution.

### Keep one branch/session

Limit one session to one branch/work item. Publication should be a workflow responsibility rather than letting the model invent branch topology.

### Externalize slow or authoritative validation

Do not ask the LLM to decide which CI job finished or whether a flaky test is fixed. The outer system launches validation and returns structured results to the agent.

For flaky tests, Dropbox repeatedly runs the test at high count, returns new failing logs, and retries up to a cap. That generalizes to:

```text
agent proposes -> system validates -> evidence returned -> agent revises
```

### Make retries bounded

Every autonomous loop needs a hard stop such as `max_iterations`, deadline, cost budget, or repeated-failure threshold.

### Integrate real engineering context

Expose logs, observability, repository-local instructions, plugins/MCP and organization-specific build tools. Avoid an AI-only parallel toolchain that does not match how engineers validate work.

## Why it is useful

The pattern separates probabilistic reasoning from deterministic control. Agents can adapt to failures without being trusted to orchestrate publication, CI state or infinite retry loops.

It also creates one reusable execution layer for interactive agents, background workflows and internal automation.

## Failure modes / caveats

- Agent-managed CI polling can stall sessions or validate the wrong target; keep CI triggering/result collection outside the model loop.
- Isolation of file state does not imply isolation of credentials/network/runtime resources.
- A common platform is only worth it when multiple workflows share execution, context and validation needs.
- Retry loops can burn cost indefinitely unless bounded.

## Prototype experiment

Wrap one existing coding-agent workflow in a small orchestrator with:

1. explicit base SHA;
2. per-run worktree/container;
3. declarative validation commands;
4. structured validation result returned to the agent;
5. max 3 repair iterations;
6. publication only after validation passes.

Measure completion rate, repair iterations, human intervention and invalid/incorrectly validated PRs.

## Sources

- https://dropbox.tech/machine-learning/introducing-nova-our-internal-platform-for-coding-agents

## Related

- [Evidence-gated coding-agent edits](evidence-gated-coding-agent-edits.md)
- [Narrow-contract background coding agent](narrow-contract-background-coding-agent.md)
