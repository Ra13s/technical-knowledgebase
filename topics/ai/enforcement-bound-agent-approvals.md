# Bind agent approvals to the exact side effect at the enforcement point

## What it is

For consequential agent actions, a UI/harness prompt such as “Approve?” is not an authorization boundary. The system that can actually execute the side effect should require an approval artifact bound to the exact operation and should revalidate it immediately before execution.

## Use when

Use this pattern for agent tools that can create consequential side effects, for example:

- deploy or release to production;
- merge code or change protected repository settings;
- send payments, purchases or refunds;
- delete or mutate production data;
- change IAM/security/network policy;
- send external communications as a user or organization.

A lightweight harness confirmation is still useful UX, but do not rely on the model, prompt or agent harness to decide whether mandatory approval applies.

## Core rule

**Approval policy belongs at the enforcement point, not inside the model loop.**

If another code path, sub-agent, plugin or prompt injection can reach the side-effecting API without traversing the confirmation UI, the confirmation is advisory rather than mandatory authorization.

## Bind approval to an action envelope

Create a canonical representation of what is being approved. At minimum bind:

```json
{
  "operation": "deploy",
  "target": "payments-api",
  "environment": "prod-eu",
  "arguments": {"version": "2026.08.29.3"},
  "requester": "user-or-delegation-id",
  "agent": "agent-instance-id",
  "expiresAt": "2026-08-29T13:10:00Z",
  "nonce": "single-use-id"
}
```

The approver should see a semantic rendering derived from this same envelope. Do not approve a vague sentence and later let the agent choose different parameters.

The authorization system should sign, MAC or otherwise integrity-protect the envelope or a digest of it. Short expiry and single-use/non-replay semantics are appropriate for high-impact actions.

## Execution flow

A reusable flow is:

```text
agent requests side effect
        |
        v
resource / tool enforcement point
        |
        +-- policy says no approval --> execute
        |
        +-- approval required ------> create exact action envelope
                                      |
                                      v
                                  human review
                                      |
                                approve / reject
                                      |
                                      v
agent retries / resumes with approval state
        |
        v
enforcement point revalidates:
  identity + policy + exact args + target + expiry + replay state
        |
        +-- mismatch --> reject / require new approval
        |
        +-- match ----> execute once + audit
```

Before execution, compare the **current** request to the approved operation. Any drift in arguments, destination, requester or policy should invalidate the approval rather than silently adapting it.

## MCP 2026-07-28 implementation shape

MCP Multi Round-Trip Requests (MRTR, SEP-2322) provide a useful pause/resume mechanism for short interactions. A tool can return `resultType: "input_required"`, plus `inputRequests` and opaque `requestState`; the client fulfills the interaction and retries the original tool call.

For authorization-sensitive use, treat `requestState` as untrusted client-controlled input and integrity-protect it. Bind it to the originating method/tool, arguments, principal and expiry.

Conceptually:

```json
{
  "resultType": "input_required",
  "inputRequests": {
    "approval": {
      "method": "elicitation/create",
      "params": {
        "mode": "url",
        "message": "Review production deployment",
        "url": "https://approve.example/requests/abc123"
      }
    }
  },
  "requestState": "<sealed-state-bound-to-request>"
}
```

On retry, the server must re-run policy and compare the retried operation with the state it approved. Opening an approval URL is not itself approval; the authorization decision should happen in the trusted approval system.

The MCP TypeScript migration guidance explicitly recommends protecting `requestState` with HMAC/AEAD and binding it to principal, originating method/parameters and expiry. SDKs may offer helpers, but the security property matters more than the specific helper.

## Audit evidence

For consequential actions, retain enough evidence to answer:

- what was requested;
- what exact representation the approver saw;
- who approved/rejected and under which policy;
- which agent/requester/delegation initiated it;
- what exact operation actually executed;
- when it executed and what result it produced.

Do not log secrets or sensitive payloads unnecessarily; store stable identifiers/hashes where full values should not be retained.

## Caveats / when not to use

- This pattern does not make an emerging protocol such as TAC/AAuth mandatory; it is a protocol-neutral authorization property.
- MCP MRTR is a lifecycle mechanism, not by itself an enterprise authorization system. You still need policy, identity, binding, replay protection and audit.
- For durable approvals that may outlive the initiating client, a persistent task/workflow model can fit better than a short MRTR retry loop.
- Do not put approval credentials or reusable high-privilege bearer tokens into model-visible context when the model does not need them.
- Approval does not remove the need for normal authorization. The approver must itself be authorized to approve that action.

## Sources

- Christian Posta, Human-in-the-loop Authorization Patterns for Agents (2026-08-24): https://blog.christianposta.com/human-in-the-loop-authorization-patterns-for-agents/
- MCP SEP-2322 Multi Round-Trip Requests: https://modelcontextprotocol.io/seps/2322-MRTR
- MCP TypeScript SDK migration guidance for securing `requestState`: https://ts.sdk.modelcontextprotocol.io/v2/migration/support-2026-07-28

## Related

- [`MCP protocol upgrades and conformance testing`](mcp-protocol-upgrades-and-conformance.md)
- [`OpenAI Programmatic Tool Calling`](openai-programmatic-tool-calling.md) — keep approval-sensitive side effects outside bounded programmatic orchestration unless an explicit trusted approval boundary exists.
