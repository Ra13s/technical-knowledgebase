# Event-reducer orchestration for bounded coding-agent PRs

## What it is

Run a small issue-to-PR agent as a **durable event-driven state machine**, not an endless model-controlled pipeline. Each model invocation is a short-lived stage with a single responsibility. A deterministic reducer maps `(persisted run state, validated event) -> next allowed action`; a separate executor performs side effects. Inkitt published a production example in August 2026, resurfaced by InfoQ in November.

This pattern complements the [deterministic agent outer loop](deterministic-outer-loop-agent-platform.md) and [durable effect claims](durable-agent-tool-effects-and-fencing.md): the new reusable unit is a **testable pure transition function** plus atomic event claiming, bounded stage detours and separate credentials.

## Use when

- Slack, Linear, webhooks and other agents can all trigger a small, low-risk coding task;
- CI finishes asynchronously and individual job webhooks may arrive more than once or out of order;
- a container can crash between editing, pushing a branch, opening a PR and receiving a response;
- the whole flow should resume without keeping a model or expensive sandbox alive while waiting;
- an automation must reject high-impact modifications instead of attempting them optimistically.

Do **not** reuse this design as justification for unattended security, payments, schema, dependency or deployment changes. Inkitt deliberately excludes such work.

## Roles, state and trust boundaries

~~~text
authenticated trigger -> dedupe + allowlist + task classification
                         |
                    durable run row
                         |
      pure reducer(state, event) -> proposed action
                         |
            transactionally claim action / lease
                         |
        executor (credentials + authoritative APIs)
                         |
     short-lived stage container (read/edit/verify only)
                         |
     inspect real diff + tests + GitHub CI status
                         |
       draft PR -> human readiness/review/merge
~~~

- **Reducer**: pure business rules for transitions; no GitHub access, no shell, model or database calls.
- **Orchestrator/executor**: owns Slack, issue tracker, GitHub PR creation and notification credentials; rechecks authoritative external state.
- **Stage containers**: one task each (specification, implementation, verification, bounded CI repair, bounded conflict repair); isolated filesystem and only short-lived repo/model access. Do not grant Slack/Linear/cloud tokens or broad egress.
- **GitHub and reviewers**: merge controls and human sign-off remain independent of the agent.

### State schema and reducer sketch

Illustrative pseudocode, **not** an Inkitt API or a drop-in framework:

~~~typescript
type Phase =
  | "INTAKE" | "IMPLEMENTING" | "VERIFYING"
  | "PUBLISHING_DRAFT" | "AWAITING_CI"
  | "READY_FOR_HUMAN" | "STOPPED";

type Run = {
  id: string;
  baseSha: string;
  phase: Phase;
  attemptVerify: number;
  attemptCiFix: number;
  attemptConflict: number;
  generation: number;       // optimistic concurrency/fencing
  branch?: string;
  prNumber?: number;
};

type Event =
  | { kind: "SPEC_APPROVED" }
  | { kind: "STAGE_FINISHED"; passed: boolean }
  | { kind: "CI_ACTIVITY" }
  | { kind: "DRAFT_PR_CONFIRMED"; number: number }
  | { kind: "RESTRICTED_DIFF" };

function reduce(run: Run, event: Event): { phase: Phase; action: string } {
  if (event.kind === "RESTRICTED_DIFF") {
    return { phase: "STOPPED", action: "report_rejection" };
  }
  if (run.phase === "VERIFYING" &&
      event.kind === "STAGE_FINISHED" && !event.passed) {
    return run.attemptVerify < 2
      ? { phase: "IMPLEMENTING", action: "repair" }
      : { phase: "STOPPED", action: "report_failed_verify" };
  }
  if (run.phase === "AWAITING_CI" && event.kind === "CI_ACTIVITY") {
    return { phase: "AWAITING_CI", action: "query_full_check_suite" };
  }
  // Other transitions must be explicitly enumerated and unit-tested.
  return { phase: run.phase, action: "no_op" };
}
~~~

Process the proposed action with a conditional write using `run_id + phase + generation`:

~~~sql
UPDATE agent_run
   SET generation = generation + 1,
       claimed_action = :action,
       claimed_at = now()
 WHERE id = :run_id
   AND phase = :expected_phase
   AND generation = :expected_generation;
-- only the winner (row count 1) may schedule the stage
~~~

For real distributed execution, use a lease/fencing token and an outbox/scheduler transaction; the reducer and claim alone cannot guarantee exactly-once delivery or cancel a stale already-running worker. An ambiguous PR or push result must be reconciled from GitHub using branch/head/task identity, never automatically repeated.

## Gates and bounded detours

1. **Intake gate:** require repo, target base SHA, clear task, affected path allowlist, maximum file count/diff size, and a low-risk change category. Reject ambiguous tasks before allocating a sandbox.
2. **Pre-edit policy:** exclude migrations/schema, shared contracts, CI/container build files, auth/payments, dependency additions/upgrades and other platform-specific high-impact paths.
3. **Post-edit gate:** compute the actual `git diff --name-status <base>...HEAD`; inspect renamed and deleted paths and ensure no path leaves the allowlist; inspect diff size/content and run tests. The agent's textual declaration is not evidence.
4. **Retries:** cap verification, CI-fix and conflict-repair independently (Inkitt used two each), plus total wall-time/cost budgets.
5. **CI completion:** on every job webhook **query GitHub's full expected check set for the exact head SHA**. A single job-completion event is not a PR-ready signal. Ignore stale job events.
6. **Publication:** create or adopt one draft PR from the run's stable branch; notify the human when staging/checks are ready. Never treat a completed model stage as human approval.

## Production evidence and caveats

Inkitt's Aug 26 account describes this architecture shipping small changes daily, but provides **no audited accepted-PR rate or controlled quality comparison**. It is a design case study, not evidence that generic one-shot changes are safe. The implementation is proprietary, so the SQL/reducer above is an illustrative reproduction of its publicly described behavior.

- A pure reducer is easy to test, but an executor with weak idempotency can still duplicate effects.
- Network isolation and model-key scope need enforcement at container/network/credential layers.
- Path checks alone do not validate semantic risk or prevent behavior changes in allowed files.
- A single orchestrator becomes a scheduling and availability dependency; persist state and use a recoverable worker.
- A draft PR that passes CI is not equivalent to a correct change; human review remains independent.

## Prototype experiment

Implement a 5-state TypeScript reducer and a SQLite/PostgreSQL run table for one documentation-only agent. Replay 100 duplicate/out-of-order webhooks; kill the stage runner before/after an external PR creation; inject a restricted rename, incomplete CI suite, stale head SHA and attempted third verification failure. Assert exactly one active branch/PR, no forbidden file writes, bounded worker attempts and a visible terminal reason. Measure duplicate-effects count, human repair time, validation cost, elapsed time and accepted PRs against a simpler linear workflow.

## Sources

- Inkitt Tech, *How we built a one-shot PR creator at Inkitt* (2026-08-26): https://medium.com/inkitt-tech/how-we-built-a-one-shot-pr-creator-at-inkitt-e1b8dbc52cda
- InfoQ, *Inkitt Builds a One-Shot PR Creator with Resumable Agent Orchestration* (2026-11-04): https://www.infoq.com/news/2026/11/inkitt-one-shot-pr-creator-agent/

## Related

- [Deterministic outer loop for coding-agent platforms](deterministic-outer-loop-agent-platform.md)
- [Durable agent tool effects and fencing](durable-agent-tool-effects-and-fencing.md)
- [Repository policy compliance gates](repository-policy-compliance-gates.md)
- [Compiled agentic workflows with Safe Outputs](compiled-agentic-workflows-safe-outputs.md)
