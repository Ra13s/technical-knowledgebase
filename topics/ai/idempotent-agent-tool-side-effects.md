# Make agent side effects idempotent in the tool contract

## What it is

A reliability rule for tool-using agents: when a mutating call may have completed even though the caller saw a timeout, disconnect or server error, retry safety should be supported by the tool/service contract rather than inferred by the model.

A September 2026 LIMBO study over 25,930 fault-injected agent episodes found two regimes. When an immediate authoritative read-back reveals the outcome, strong models can often recover by verify-then-decide. When the original request may still be in flight or the transport can redeliver it, reasoning alone cannot guarantee exactly-once. Stable idempotency keys are the reusable primitive.

## Use when

Use this for agent tools that create externally visible or persistent state: publishing artifacts, creating work items, starting jobs, changing resource configuration, writing records, or invoking batch operations.

The more costly a duplicate logical action is, the less acceptable "the model will check first" becomes.

## Tool contract

For a non-idempotent write, include a stable logical operation key:

```json
{
  "operation": "publish_artifact",
  "idempotency_key": "run-123:publish:artifact-42",
  "arguments": {
    "artifact": "artifact-42"
  }
}
```

The service should execute one logical operation at most once per key, persist key-to-outcome state, return the original result on replay, and keep the key stable across retries, resumes and agent handoffs.

Do not generate a fresh key such as `-retry1` for an uncertain retry.

## Implementation rules

### Generate the key outside free-form model reasoning when possible

A tool gateway/harness can derive the key from stable workflow identity:

```text
idempotency_key =
  hash(workflow_id, logical_step_id, target, semantic_arguments)
```

The agent may choose whether to retry, but the infrastructure should preserve identity for the same logical action.

### Expose a status/read-back operation for every consequential write

Prefer a pair such as:

```text
create_artifact(..., idempotency_key)
get_artifact_status(idempotency_key | operation_id)
```

Document whether read-back is strongly or eventually consistent, how long an original request may remain in flight, batch resume semantics, and how long idempotency records are retained.

An eventually consistent read returning "not found" is not proof that a timed-out write did not happen.

### Classify unknown outcomes before retrying

```text
lost acknowledgement / misleading server error
  -> authoritative read-back may resolve outcome

partial batch
  -> status + resumable/idempotent batch contract

late commit / request still in flight
  -> verification may observe "absent" too early
  -> stable key or explicit escalation required

transport redelivery
  -> service-side idempotency prevents duplicate effect
```

### Disable transparent retries for unsafe writes

Generic client retry middleware must not blindly retry a non-idempotent mutation after timeout or server error unless the request carries a stable idempotency key or the operation is otherwise idempotent.

### Bind approval and idempotency to the same logical action

For approval-gated effects, the approved action envelope and the idempotency identity should describe the same semantic operation. A retry of the same approved operation can reuse the key. Parameter drift creates a different logical operation.

### Make batch semantics explicit

For multi-item writes, define whether the key covers the whole batch atomically, each item, or a resumable batch cursor.

### Grade against committed effects, not agent self-report

In tests and production audits, compare expected logical operations with the service's authoritative ledger/audit state rather than agent self-report.

### Fault-inject the contract

Test response loss after commit, server error after commit, late commit after timeout, duplicate delivery, partial batch, stale status reads, and agent/harness restart between attempt and retry. The same logical key should survive every retry/resume path.

## Why it is useful

Exactly-once is a distributed-systems property with an information boundary. If the client cannot observe whether an operation is still in flight, a model cannot reason its way around that missing fact.

> If duplicate side effects matter, design idempotency into the mutating tool before tuning the agent's retry prompt.

## Evidence and caveats

LIMBO is a September 2026 preprint with a deterministic simulated service benchmark, not a production incident study. Its exact duplicate percentages are workload-specific.

The stronger result is structural: for late commits without a known in-flight bound, verify-then-retry cannot distinguish "did not happen" from "will happen later". Under the paper's keys-everywhere contract, reusing the same key eliminated duplicates in the measured re-issued late-commit cases; failures remained when keys were omitted or changed.

Idempotency still needs correct server implementation, retention policy and semantic key design.

## Prototype experiment

Wrap one state-changing internal agent tool: add `idempotency_key` and `get_status(operation_id)`; derive the key in the harness from stable workflow/step identity; persist key-to-result server-side; disable transparent retry when no key exists; inject late-commit, lost-ack, redelivery and partial-batch faults; restart the agent between attempts; and assert the authoritative effect ledger contains exactly one logical effect.

Track duplicate effects, abandoned required effects, escalations and recovery latency.

## Sources

- Li, *Where Does Exactly-Once Live? Model, Harness, and Tool-Contract Effects on Duplicate Side Effects in LLM Agents* (2026-09-24): https://arxiv.org/abs/2609.29095

## Related

- [Bind agent approvals to the exact side effect](enforcement-bound-agent-approvals.md)
- [Deterministic outer loop for coding-agent platforms](deterministic-outer-loop-agent-platform.md)
- [Evidence-gated coding-agent edits](evidence-gated-coding-agent-edits.md)
