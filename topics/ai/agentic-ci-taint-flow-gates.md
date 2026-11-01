# Audit agentic CI as a taint graph from untrusted text to privileged sinks

## What it is

An AI-powered GitHub Actions workflow has **two distinct injection boundaries**:

- **Prompt-to-Agent (P2A):** an attacker-controlled issue, PR description, comment, commit message or linked file reaches the agent's prompt/context; the agent then uses privileged tools.
- **Prompt-to-Script (P2S):** attacker-controlled text influences model-generated output, which a later Actions step treats as a shell command, path, GitHub expression, API argument or privileged release input.

Treat LLM outputs as *tainted*, even if the model returned JSON. A schema confirms shape, not permission or trustworthiness. Audit the complete source -> prompt -> generated output -> downstream sink chain. Neither tool permission gating nor final-diff scanning alone detects all these paths.

A May 2026 empirical study introduced TaintAWI, analyzing 13,392 agentic workflows across 10,792 repositories. It reported 519 candidate vulnerabilities, of which 496 met its exploitability test (95.6% precision) and 343 were previously unknown. These are author-reported results under a stated threat model, not an independently reproduced detection rate; the paper's disclosure responses were limited. The data support prioritizing the failure mode, not assuming a particular analyzer will work unchanged in our repository.

## Use when

- an Actions workflow sends issue/PR text to Codex, Claude, Copilot, Gemini or a custom LLM;
- agent outputs become environment variables, step outputs, shell interpolations or GitHub API mutations;
- the workflow runs on untrusted fork PRs, outside collaborators, comments or `pull_request_target`;
- a coding agent can create a PR, run scripts or access package-publishing credentials;
- reviewing new agentic actions or tool integrations in CI.

## Concrete audit method

Construct a dependency graph from the GitHub Actions YAML, referenced scripts, reusable actions and the model tool wrapper:

~~~text
Sources:
  github.event.issue.body / comment.body / pull_request.body
  PR head files / issue-linked documents / fetched web and MCP content
  generated assistant text (derived tainted source)
       |
       v
Prompt boundary:
  with.prompt, stdin, environment variables, agent session input
       |
       v
Model / agent execution:
  shell, write_file, GitHub MCP, safe outputs, third-party tools
       |
       v
Sinks:
  shell eval or command substitution
  workflow dispatch / branch pushes / PR creation
  artifact publication / secrets access / package release
  host tool or network exfiltration
~~~

### Rules to enforce

1. **Tag origins and derived results.** An output of a model call stays untrusted unless independently constrained to a fixed enum or mapped to a preapproved action. The model is not a sanitizer.
2. **Remove raw interpolation.** Never put an untrusted `github.event.issue.body` (or an LLM-generated output) directly inside a `run:` script. Instead pass untrusted event text through a quoted environment variable or file and treat it as *data*, not shell instructions. Quoting is necessary but not sufficient: never feed that data into `eval`, shell flags, `bash -c`, template engines or another interpreter.
3. **Constrain P2A capabilities.** Use sandboxed, read-only agent tools by default; authorize writes only in a separate stage that validates specific operation schemas, repository/path allowlists, count limits and base SHA.
4. **Constrain P2S outputs.** Parse model JSON against a strict schema, reject unknown fields, check enumerated decisions in trusted code and construct the actual command/API call from trusted literals. Do not execute a model-supplied command string.
5. **Cross-check workflow privilege context.** Pay special attention to `pull_request_target`, token scopes, fork checkouts, inherited environment secrets, cache/artifact writes and reusable action pinning. A low-privilege issue author's text must not silently gain high-privilege write capabilities.
6. **Fail closed on indeterminate provenance.** If an action wrapper or dynamic script cannot be analyzed, mark it for review rather than silently assuming it has no path to a sensitive sink.

### Defensive example (non-executable agent output)

An agent may suggest a decision as data:

~~~json
{"decision":"needs-docs","target":"main","summary":"New API endpoint"}
~~~

A separate trusted step must validate `decision` against a fixed enum, hard-code/allowlist `target`, construct a PR request using an allowlisted file diff and retain the source SHA. It must *not* accept an arbitrary `command`, `script` or unrestricted `repository` field. An invalid output produces a blocked result, not a best-effort shell call.

## Testing

Write a regression matrix with:
- benign issue/PR content -> one permitted suggestion;
- untrusted issue text containing instructions to modify unrelated files -> no privileged mutation;
- malformed JSON and an extra `script` property -> schema reject;
- output containing an unexpected branch or repo -> authorization reject;
- a fork PR under `pull_request_target` -> no write token or secrets in agent job;
- replay of a legitimate event -> at most one logical write under a stable dedup key.

Capture actual GitHub Actions permission scopes, tool calls and resulting GitHub objects as the oracle. Static flags should lead to review; they are not proof of exploitability.

## Failure modes and evidence limits

- YAML-only scanning misses scripts, composite actions, secret handling in containers and dynamic helpers. Model-derived data may cross multiple action-output and artifact boundaries.
- Restricting model tool calls is not enough if a later shell step uses generated text unsafely.
- Treat YAML expression escaping and shell quoting as different problems. Use checked-in scripts and typed input parsers when possible.
- The TaintAWI paper demonstrates a threat-model-specific benchmark, not a public production adoption measure; independently verify any third-party scanner before using it to fail all PRs.
- A safely escaped input can still ask the agent to do the wrong thing; the write executor must re-check authorization outside the model.

## Prototype experiment

Inventory `.github/workflows/*.yml` and any referenced action/agent definitions. Build a small static linter that tags `github.event.*.body`, model outputs and retrieved repo content as tainted; trace references through `with`, `env`, `steps.*.outputs` into `run` and write-scoped tools. Start with warning-only mode and a small set of confirmed positive/negative fixtures. Compare findings with a manual review, patch real unsafe flows and keep regression examples in CI.

## Sources

- https://arxiv.org/abs/2605.07135
- https://github.blog/ai-and-ml/generative-ai/under-the-hood-security-architecture-of-github-agentic-workflows/
- https://github.github.com/gh-aw/reference/threat-detection/
- https://aws.amazon.com/blogs/security/balancing-speed-and-safety-a-control-framework-for-ai-coding-agents/

## Related

- [Compiled agentic workflows and safe outputs](compiled-agentic-workflows-safe-outputs.md)
- [Environment-first agent containment](environment-first-agent-containment.md)
- [Repository policy compliance gates](repository-policy-compliance-gates.md)
- [Agent harness supply-chain gates](agent-harness-supply-chain-gates.md)
