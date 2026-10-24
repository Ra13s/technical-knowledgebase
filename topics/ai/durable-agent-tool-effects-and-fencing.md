# Crash-recoverable coding agents: claim tool effects, fence stale workers

## What it is

Recover a coding-agent session across crashed workers without automatically repeating a tool operation that might already have changed GitHub, a database or an external API. Temporal's experimental Pi integration combines durable session transcripts, workspace snapshots, per-tool claims and fenced writes.

## Use when

Long-running agents can outlive one runner, execute non-idempotent tools, or resume from ambiguous timeouts. This adds the missing recovery contract to a deterministic agent outer loop; it does not provide exactly-once external side effects.

## Implementation shape

~~~text
Workflow engine: task/turn/step lifecycle, deadlines and retry budgets
Durable store: transcript + file snapshots + tool claims/results
Worker: isolated step execution and tool calls
Authority: external API, remote Git ref or database

runModelCall -> runToolCall (each tool) -> sealStep -> next step
~~~

1. Give each logical call an identity `session/turn/step/tool_call`.
2. Before invocation, atomically persist a claim and request digest. If a saved result exists, return it without execution.
3. After invocation, persist the result before advancing the transcript. A retry finding only a claim must return **outcome unknown**, not invoke the tool automatically.
4. The agent must inspect the authoritative system. After an interrupted PR creation, look for a PR on the intended head branch; after a push, fetch the remote ref. Adopt confirmed work, retry only when absence and retry safety are established, or stop for manual reconciliation.
5. Bind transcript appends to a lease/fencing token and reject writes from a timed-out, superseded worker. A timeout alone does not kill its old process or undo external effects.
6. Keep the conversation in a durable transcript. Save workspace changes separately; Pi's prototype ships project snapshots as Git bundles beside the session file, without modifying the project's own Git history. On takeover, sync both.
7. Only a designated step-sealing operation writes ordered tool results. Keep tools for a step on a coherent worker and enforce time/token budgets outside the model.

### Tool state machine (conceptual)

~~~text
UNSEEN -> CLAIMED -> RESULT_RECORDED -> SEALED
             |
             +---- crash / lost response -> OUTCOME_UNKNOWN
                                               |
                                               -> reconcile external truth
~~~

The claim means *may have started*, not *did happen*. For a retryable model call, first check whether its answer is already in the transcript; only send another model request when no answer was saved.

## Verification experiment

Clone `https://github.com/temporalio/pi-temporal`; run its documented multi-worker chaos demo, which kills workers mid-task. For our own harness, inject deaths before the claim, after the claim, after the external effect and after saving the result. Assert that interrupted effects are not silently replayed, recorded results are reused, stale workers cannot append, and external state is inspected before reissue.

Test an interrupted `git push` where the remote ref already advanced even though the runner never received success. The recovered agent must detect and adopt the existing commit, not invent a second push.

## Evidence and caveats

Temporal reports that its October 2026 prototype survives injected worker deaths and completes coding tasks. It depends on coherent shared storage and correct fencing; this is not a production guarantee for arbitrary filesystems or clocks. Transcript fencing does not prevent an old worker's already-started network call. When the API supports idempotency keys, use them as well. Treat ambiguous irreversible operations as requiring human reconciliation.

## Sources

- https://temporal.io/blog/the-immortal-life-of-pi-running-the-pi-coding-agent-on-temporal
- https://github.com/temporalio/pi-temporal

## Related

- [Idempotent side effects](idempotent-agent-tool-side-effects.md)
- [Deterministic agent outer loop](deterministic-outer-loop-agent-platform.md)
- [Authoritative telemetry](external-authoritative-agent-telemetry.md)
