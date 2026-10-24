# Enforce agent effects with a sandbox and cross-tool temporal policies

## What it is

A coding-agent sandbox should enforce both **where execution can reach** and **what an operation may do**, independently of the model's instructions. AWS's October 2026 developer-preview **Strands Box** is a concrete OSS implementation: macOS Seatbelt restricts direct OS access; a trusted gateway, Shell/Python interpreters and MCP broker emit normalized events; **Dogwood** `permit`/`forbid` policies decide operations using the request and earlier event history.

The reusable pattern is *OS boundary + outside-the-agent enforcement + event-history-aware policy*, not a requirement to adopt Box specifically.

## Use when

- a background agent executes shell/Python, invokes MCP servers, touches local repositories or makes network requests;
- an agent needs access to a service but only a narrow operation on it (e.g. read status, never change deployment);
- rules depend on order or frequency, such as "at most three successful notifications in ten minutes";
- credentials must be injected into permitted requests without reaching agent-visible environment variables or context.

## Implementation architecture

~~~text
untrusted model / agent process
    |               |
    |               +--> OS sandbox: direct files/processes/network
    v
trusted shell / Python / MCP broker / egress proxy
    -> normalize {actor, operation, args, previous events}
    -> authorization engine (permit + forbid + temporal history)
         -> reject with rule ID + reason
         -> allow effect; record request and actual response
    -> credential broker adds scoped credential only after policy
    -> destination API / filesystem / tool
    -> protected decision and executor telemetry
~~~

Three boundaries are distinct: OS containment prevents bypass of the environment's maximum reach; policy enforces operation-level semantics **only at instrumented interception points**; downstream IAM/API authorization still restricts the service account.

### Minimal preview example (macOS Apple silicon)

The following illustrates the structure of the public Box example. Paths and tool locations must match the host; keep secret storage outside the workspace.

`box.toml`:

~~~toml
name = "read-only-investigator"
box_dir = "/Users/me/.box/investigator"
policy = "policy.dw"

[agent]
command = ["strands", "--builtin-tools", "shell"]
workspace = "/Users/me/investigation"

[agent.filesystem]
read = ["/Users/me/investigation"]
write = ["/Users/me/investigation"]

[egress.model]
destinations = ["api.anthropic.com"]
secret.ref = "env://ANTHROPIC_API_KEY"
~~~

`policy.dw` (a minimal, deliberately narrow example):

~~~text
@id("allow_model")
permit (principal, action == Box::Action::"http:request", resource)
when {
  context.input.host == "api.anthropic.com" &&
  context.input.port == 443
};
~~~

Launch with `box run --config box.toml` after following the repository's setup guide. A real deployment needs explicit permits for every required tool/operation and verified defaults; a credential binding alone does **not** imply network permission.

For historical policy, Dogwood supports `when temporal { ... }` conditions on events from multiple tool surfaces. The published example permits Slack `POST /api/chat.postMessage` but forbids a fourth post once **three HTTP 200 response events** have occurred in the previous ten minutes. Counting successful **responses** avoids counting rejected calls; counting **attempts** would be a different policy. Preserve both request and response identities and test partial failures.

### Acceptance/fault tests

1. Attempt `rm` outside the workspace, a shell command not on the allowlist, and an HTTP POST to an otherwise permitted host; assert they are rejected at the appropriate boundary.
2. Read a sensitive file via one interpreter and then try an egress call via another; verify a history-dependent rule fires only if that file read actually passed through an instrumented boundary.
3. Exercise allowed API calls using only a placeholder credential; inspect process environment, logs and tool output for any real secret exposure.
4. Run four successful notification posts inside ten minutes, then repeat with an API 500 and a denied request. Confirm the specified success/attempt semantics.
5. Restart the box and try concurrent agents: check whether history survives and whether any desired **global** budget is actually enforced; for cross-run quotas use a central rate limiter, not only a local history.

## Why useful / evidence

Box provides one policy language across shell, interpreted code, HTTP and MCP tool calls; it also separates credential custody from the untrusted agent and exposes refusal IDs to make recovery intelligible. Its OSS repository includes integration and native-filesystem tests, so we can experimentally verify specific claimed boundaries rather than trust a diagram.

## Failure modes / compatibility

- **Preview and platform constraint (October 2026):** Box supports **macOS 15+ on Apple silicon**, not Windows or Linux production use. Its setup example also expects specific Node and model-provider versions. On Windows/Linux, adopt the architecture with an existing container/microVM plus trusted proxy, or use a macOS test machine; do not claim Box itself runs there.
- **Coverage hole:** direct filesystem grants in `box.toml` use OS isolation but do **not** produce individual Dogwood file events. Temporal policies cannot observe actions that bypass instrumented surfaces; keep sensitive paths out of direct grants.
- **Trusted computing base:** interpreters, proxy and broker live outside the sandbox. Their correctness and host configuration are security-sensitive.
- **Local history is not global policy:** cross-session rate limits and expensive effects need centrally authoritative counters and tool-side idempotency.
- **Allowed destination != safe action:** constrain method/path/tool/schema and downstream identity, not merely hostname. A 200 status is not proof a business side effect committed.
- **No security proof:** a developer preview and illustrative examples do not establish resistance to all sandbox escapes.

## Prototype experiment

On a supported macOS host, pin the Box source revision, add one read-only GitHub investigation task and synthetic adversarial MCP server. Verify exact allow/deny decisions, external credential custody, action history, restart behavior and instrumentation gaps. Compare baseline success, overhead, block accuracy and missing-event rate. For a non-macOS runner, prototype the same enforcement contract behind a restricted egress/MCP gateway.

## Sources

- AWS (2026-10-07), Strands Box launch: https://aws.amazon.com/blogs/opensource/introducing-strands-box-ai-agent-sandboxes-powered-by-dogwood/
- Source, tests and current constraints: https://github.com/strands-agents/box
- Dogwood language and temporal rules: https://dogwood-policy.github.io/dogwood/

## Related

- [Contain coding agents at the environment boundary first](environment-first-agent-containment.md)
- [Bind approvals to the actual side effect](enforcement-bound-agent-approvals.md)
- [Externalize authoritative agent telemetry](external-authoritative-agent-telemetry.md)
- [Make agent tool effects idempotent](idempotent-agent-tool-side-effects.md)
