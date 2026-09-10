# Contain coding agents at the environment boundary first

## What it is

A security pattern for tool-using/coding agents: make filesystem, process, network and credential limits deterministic properties of the execution environment, then use model-level classifiers/prompts and human approvals as additional defense-in-depth rather than the final boundary.

The model can still be useful and well-behaved inside the perimeter. The perimeter defines the maximum blast radius when the model, user, prompt, tool output or remote service is wrong or malicious.

## Use when

Use this whenever an agent can:

- run shell commands or generated code;
- read a developer or worker filesystem;
- call the network;
- access credentials or connectors;
- operate unattended or in parallel;
- consume untrusted repository/web/MCP content.

Increase isolation strength as user oversight becomes less reliable or agent autonomy increases.

## Core rule

**Enforce containment in the environment; steer behavior in the model.**

```text
untrusted task / repo / tool output
              |
              v
        model reasoning
              |
              v
   sandboxed execution boundary
      |       |        |
      |       |        +-- scoped identity/credentials
      |       +----------- egress policy
      +------------------- filesystem/process policy
              |
              v
      allowed side effects only
```

Prompts, classifiers and confirmations can reduce risky attempts. They should not be the only thing preventing access to data or capabilities the agent never needed.

## Practical controls

### Keep credentials out of the sandbox when possible

Prefer host-side credential custody and issue the agent/session a scoped-down, revocable token for only the capability it needs.

```text
user credential -> host keychain / broker
                        |
                        v
              per-session scoped token
                        |
                        v
                   sandbox
```

If `~/.aws/credentials` is not mounted or visible, an injected prompt cannot exfiltrate it by finding a clever shell command.

### Default network deny, then grant capabilities deliberately

A domain allowlist is not merely a destination list. It grants every reachable operation on that destination unless the proxy/API boundary further constrains it.

For sensitive APIs, bind egress to credentials provisioned by the sandbox or trusted broker. Reject attacker-supplied API keys/tokens even when the hostname itself is allowed.

```text
sandbox -> egress proxy -> allowed API
             |
             +-- destination allowed?
             +-- request uses expected scoped identity?
             +-- dangerous server-side-fetch headers blocked?
```

### Resolve symlinks before validating filesystem paths

If a mounted/allowed directory is a trust boundary, canonicalize/resolve symlinks first and then check that the resolved path remains inside the allowed root.

```text
requested path
   -> resolve/canonicalize
   -> verify inside allowed mount
   -> apply read/write/delete policy
```

Checking the textual path first allows an in-tree symlink to point outside the approved workspace.

### Establish trust before loading project-local executable configuration

Treat repository open/config load as an inbound trust transition. Do not execute hooks or parse privileged project-local configuration before the repository/folder has been accepted as trusted.

This applies beyond one client: startup hooks, local plugins, task definitions and localhost listeners can all become pre-consent execution paths if “local” is assumed to mean trusted.

### Separate the reasoning process from risky execution when useful

The agent loop does not necessarily need to run inside the same VM/container as generated code. Keeping code/shell execution behind the hard boundary can preserve containment while allowing the control plane to remain responsive if the sandbox fails to start or crashes.

### Budget for observability lost to isolation

VM/container boundaries can hide activity from host EDR or ordinary process telemetry. Decide early how security events, process/network activity and audit evidence escape the sandbox safely — for example via controlled OpenTelemetry export or host-side proxy logs.

## Match containment to the user and workflow

A developer who understands shell commands may tolerate an OS sandbox plus occasional approvals. An unattended migration agent or non-technical user should not depend on correctly evaluating every command prompt.

A useful decision rule:

```text
less reliable human oversight OR more autonomy
                    => stronger always-on environment boundary
```

Reducing approval prompts can be a benefit of stronger sandboxing, not a reduction in safety. Anthropic reported an 84% reduction in permission prompts after adding an OS-level sandbox to Claude Code.

## Treat remote tool output as mutable untrusted input

Install-time review is strongest for local, pinned tools. A hosted MCP server or cloud connector can change after approval, and even an audited connector can return attacker-controlled content.

Therefore separate:

- **code/dependency trust** — pin/review the tool itself where possible;
- **content trust** — treat live tool results as potentially hostile prompt input;
- **capability trust** — constrain what executing on that content can actually reach.

## Why it is useful

Model-level safety is probabilistic and human approvals degrade under fatigue. Anthropic describes incidents where project configuration executed before the trust dialog, a controlled phishing prompt exfiltrated credentials in 24 of 25 retries, and a hostname allowlist still allowed files to be uploaded to an attacker-controlled account on an approved API domain.

Those are different failures with the same architectural answer: a hard environment boundary should remain effective even when intent classification, the model or the user makes the wrong call.

## Caveats

- Sandboxing does not remove the need for normal authentication, authorization, secret handling, CI and human review.
- Stronger isolation has operational cost: startup latency, filesystem semantics, tool compatibility and security visibility can all get worse.
- Avoid custom isolation primitives when mature VM/container/syscall mechanisms can provide the boundary; custom glue is often the least-tested layer.
- Egress policy must model capabilities and identity, not only DNS names.
- A sandbox protects resources outside its boundary; the writable workspace inside it can still be damaged.
- Remote MCP/tool results remain a prompt-injection surface even if the transport/tool implementation is trusted.

## Prototype experiment

For one background coding-agent workflow:

1. run code/shell in a disposable container or VM;
2. mount only the repository worktree, with explicit read/write policy;
3. keep developer credentials outside and inject a scoped short-lived token only when needed;
4. deny outbound network by default and allow only required APIs;
5. canonicalize paths before mount/write authorization;
6. verify project hooks/config do not execute before repository trust is established;
7. export audit/process/network events through a controlled channel;
8. red-team with a repository file that asks the agent to read a host secret and POST it externally.

The experiment passes when the agent may attempt the action but the environment prevents both secret access and unauthorized egress.

## Sources

- Anthropic, *How we contain Claude across products* (2026-05-25): https://www.anthropic.com/engineering/how-we-contain-claude

## Related

- [Gate coding-agent harness configuration as a supply chain](agent-harness-supply-chain-gates.md)
- [Bind agent approvals to the exact side effect](enforcement-bound-agent-approvals.md)
- [Deterministic outer loop for coding-agent platforms](deterministic-outer-loop-agent-platform.md)
- [Evidence-gated coding-agent edits](evidence-gated-coding-agent-edits.md)
