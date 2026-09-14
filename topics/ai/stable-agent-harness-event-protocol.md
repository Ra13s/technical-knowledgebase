# Expose one agent harness through a stable event protocol

## What it is

A boundary pattern for agent products: keep the agent loop, thread persistence, tool execution, approvals and configuration in one harness runtime, then expose that runtime to IDEs, CLIs, desktop apps, web frontends and remote clients through a **small, versioned, bidirectional event protocol**.

OpenAI's Codex App Server is a production example. Codex web, IDE and app surfaces share the same harness, while the protocol translates internal runtime events into a smaller client-facing lifecycle around threads, turns and typed items.

The reusable idea is not JSON-RPC specifically. It is to prevent each client surface from becoming its own slightly different agent implementation.

## Use when

Use this when the same coding or operational agent needs to appear in more than one client or execution environment, especially when clients need:

- streaming progress and model output;
- diffs or other typed artifacts;
- pause/resume and reconnection;
- server-initiated approval requests;
- long-running work that must survive UI disconnects;
- independent client and harness release cycles.

For a one-shot CI task, a CLI/SDK is usually simpler. For exposing an agent merely as a callable tool, MCP may be enough. Do not build a rich harness protocol unless the lifecycle actually needs it.

## Protocol shape

A practical minimum is:

```text
client
  -> initialize(capabilities, client version)
  -> thread/start | thread/resume | thread/fork
  -> turn/start(input)

server
  -> item/started
  -> item/*/delta ...
  -> item/completed
  -> approval/request  <--- server-initiated request
  -> turn/completed
```

Keep durable concepts explicit:

- **Thread** — resumable conversation/task state.
- **Turn** — one bounded user/agent execution within a thread.
- **Item** — typed atomic event/artifact such as message, tool execution, approval or diff.

The exact vocabulary can differ. What matters is that lifecycle and identity are protocol concepts rather than UI conventions.

## Implementation rules

### 1. Let the harness own durable task state

A browser tab, IDE panel or terminal process is not a reliable source of truth for a long-running task.

```text
client state = render/cache/control surface
server state = durable thread + event history + execution state
```

Persist enough state for a new client session to reconnect and catch up after a disconnect.

### 2. Translate runtime internals at the boundary

Do not expose every internal model/tool/runtime event directly. Add a translation layer that converts unstable implementation details into a small stable external event set.

This lets the harness change compaction, model providers, tool internals or execution details without forcing every client to update in lockstep.

### 3. Make the channel bidirectional

Agent execution is not purely request/response. The runtime may need a human decision during a turn.

Model approval as a server-initiated request with an explicit response:

```text
server -> approval/request(id, action, arguments, evidence)
client -> approval/response(id, allow | deny)
```

The turn remains paused until the decision is resolved. Keep the actual side-effect enforcement in the execution boundary, not only in the UI.

### 4. Negotiate capabilities during initialization

Start each connection with an explicit handshake carrying client/server identity, versions and optional capabilities.

Use capability flags for experimental or optional protocol features rather than assuming every client updates simultaneously.

```json
{
  "method": "initialize",
  "params": {
    "client": {"name": "ide-plugin", "version": "2.4.0"},
    "capabilities": {"experimentalArtifacts": false}
  }
}
```

### 5. Generate client bindings from one schema

Maintain the protocol schema next to the harness and generate language bindings or JSON Schema from it. Contract-test generated clients against the server.

Codex exposes TypeScript and JSON Schema generation from its protocol. The broader rule is: do not hand-maintain subtly different protocol models in every client language.

### 6. Choose release coupling deliberately

Two reasonable modes are:

- **bundled/pinned** — a local client ships the exact harness binary it tested;
- **decoupled** — clients may talk to newer harness versions, requiring backwards-compatible protocol evolution.

Do not accidentally combine the second deployment model with the compatibility assumptions of the first.

### 7. Separate protocol from transport

Keep lifecycle semantics stable while allowing transports to differ by environment:

```text
local IDE/app     -> stdio / local socket
hosted runtime    -> tunneled persistent connection
remote workstation -> authenticated encrypted transport
```

Treat remote transports as security boundaries: authenticate peers, encrypt traffic, bind listeners narrowly and handle backpressure/retries explicitly.

### 8. Use the smallest integration surface that preserves semantics

Decision rule:

```text
one-shot automation       -> CLI / SDK
agent as callable tool    -> MCP
rich first-party UI       -> full harness event protocol
cross-provider common UI  -> common protocol if its semantic subset is enough
```

Do not force provider-specific concepts such as rich diff streaming, specialized approvals or session control through a lowest-common-denominator protocol if doing so destroys behavior the product needs.

## Why it is useful

Without this boundary, every new client tends to reimplement parts of thread state, approvals, event mapping and tool behavior. That produces drift and makes harness improvements expensive to propagate.

A stable event protocol instead gives one place for agent semantics and many replaceable presentation surfaces. It also makes remote execution and reconnectable long-running work much easier to reason about.

## Failure modes and caveats

- **Leaking internal events:** clients become coupled to implementation details and protocol evolution freezes.
- **UI-owned state:** reconnects create split-brain task history.
- **Unbounded event queues:** slow clients can exhaust server memory; use bounded queues/backpressure and retry semantics.
- **Approval theatre:** a protocol approval event is not sufficient unless the side-effect layer enforces the decision.
- **Version ambiguity:** independently released clients need explicit compatibility policy and conformance tests.
- **Protocol overkill:** one-off automation does not need a long-lived event server.

## Prototype experiment

For one existing agent currently embedded directly in a CLI or service:

1. extract thread/turn/item identities;
2. expose `initialize`, thread lifecycle and turn lifecycle over local JSONL or another simple bidirectional transport;
3. translate internal events into a deliberately small external schema;
4. add one server-initiated approval flow;
5. persist enough event history to reconnect a second client mid-task;
6. generate one client binding from the schema;
7. run compatibility tests with an older client against a newer server.

If the protocol stays small while two clients can share one harness without behavior drift, the boundary is paying for itself.

## Sources

- https://openai.com/index/unlocking-the-codex-harness/
- https://developers.openai.com/codex/app-server/

## Related

- [Deterministic outer loop for coding-agent platforms](deterministic-outer-loop-agent-platform.md)
- [Bind agent approvals to the exact side effect](enforcement-bound-agent-approvals.md)
- [MCP protocol upgrades and conformance](mcp-protocol-upgrades-and-conformance.md)
