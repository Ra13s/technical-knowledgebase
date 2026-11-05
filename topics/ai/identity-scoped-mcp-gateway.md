# One identity-scoped MCP gateway for people and autonomous agents

## What it is

Expose an organization's operational MCP tools behind **one identity-aware gateway** rather than giving each engineer or agent a separate collection of local MCP servers and vendor credentials. Authenticate humans and service agents differently, authorize **each tool call**, execute tool servers in the appropriate environment, and keep downstream credentials at the trusted gateway/runtime boundary.

Eon's October 2026 production deployment uses Amazon Bedrock AgentCore Gateway, Cognito, Cedar policies, AgentCore Runtime and Identity. The architecture and the critical **tool-catalog scale trade-off** transfer to other OAuth/OIDC + gateway platforms; the AWS configuration itself is product-specific.

## Use when

- CLI/IDE agents need tools spanning GitHub, Kubernetes, metrics, tickets and cloud systems;
- a team struggles to rotate per-developer API tokens or install/update many local MCP servers;
- an autonomous agent needs a different permission set from a human doing the same investigation;
- tool manifests become so large that policy compilation, startup or model context is affected;
- environments/accounts must remain isolated, especially production.

## Implementation blueprint

~~~text
human IDE -- OIDC auth-code + PKCE / corporate SSO --> signed user JWT
autonomous bot -- OAuth client_credentials -----------> service JWT
                                     |
                           single remote MCP endpoint
                                     |
                       validate issuer/audience/expiry
                      -> principal + roles + environment
                      -> policy check for exact tool + arguments
                      -> discover only appropriate tools
                                     |
                  route to isolated target (K8s / GitHub / Grafana)
                      -> target-scoped service credential / workload identity
                      -> execute and audit result
~~~

For a reference AWS implementation, Eon federates corporate SSO through Cognito for humans and authenticates bots with separate client credentials. A managed MCP gateway applies **Cedar in ENFORCE mode**. MCP runtimes live in separate AWS accounts and access is arranged with IAM trust rather than per-target VPC peering. The gateway/Identity vault supplies vendor service credentials **only at execution**.

### Authorization contract

- Assign a distinct principal to each unattended agent; never let a bot borrow an engineer's broad SSO session.
- Require an explicit permit for every target tool, environment and risk tier; enforce again at `tools/call`, independent of what `tools/list` displays.
- Production investigation tools are read-only by default; allow low-impact writes such as issue/PR comments only to designated principals. Deployment, deletion or privileged changes need separate approval/authorization.
- Make authorization policy versioned code with reviewed PRs, test cases and audit evidence.
- Retain stable session/run ID and principal on the gateway, policy decision, tool invocation, credential use and result. Scope downstream target token separately from the caller's gateway JWT.

A simplified authorization policy **sketch** from Eon's architecture (adapt to the target gateway's real Cedar schema):

~~~cedar
permit(
    principal == Agent::"sre-bot",
    action == Action::"tools/call",
    resource == Tool::"github_add_pr_comment"
);
forbid(
    principal in Group::"humans",
    action == Action::"tools/call",
    resource in ResourceGroup::"write-tools"
);
~~~

This is a *schema-dependent illustration*, **not** paste-ready for an arbitrary AgentCore Gateway. Validate real principal/action/resource namespaces and run allow/deny tests before enabling ENFORCE mode.

### Tool catalog: indexed vs dynamic

Eon hit a non-obvious constraint: **pre-indexing every MCP tool schema inflates the Cedar policy**, ultimately nearing the service quota.

- Use **DEFAULT / pre-indexed** mode for smaller targets where semantic tool search or per-user OAuth consent depends on indexed schemas.
- Use **DYNAMIC** mode for large catalogs that can answer `tools/list` at request time; this avoids pre-indexing schemas in policy but **loses semantic tool search for those targets**.
- Use a dedicated target-side wrapper to filter exposed tools by workflow/role and translate credentials when the upstream MCP server is too broad. Test what happens when a newly added tool is discovered before policy review.
- This is **capability discovery**, not authorization. Treat the resulting manifest as untrusted and check every call.

### Client authentication compatibility

Eon had to implement an OAuth compatibility shim because its clients expected standard OAuth `state` (CSRF protection) while the managed service used session-URI binding. A shim must preserve identity, state integrity, redirect allowlists, expiry and session association. Never solve a handshake mismatch by disabling `state` validation.

## Schema-derived discovery with owner-reviewed activation (Uber)

Uber's October 1, 2026 MCP Gateway is a second production design, complementary to Eon's identity and credential model. It derives virtual MCP tool definitions from an existing **IDL registry**, rather than requiring every team to publish a native MCP server:

~~~text
IDL registry (Protobuf / Thrift) + native server heartbeats
  -> Cadence-backed AutoCrawler on a schedule
  -> extract method schema + comments / native tools/list
  -> generate JSON tool description and source adapter
  -> upsert DISABLED in registry
  -> owning team reviews schema + AI-written description + diff
  -> approve / activate / version / rollback
  -> runtime gateway translates MCP <-> HTTP/gRPC/TChannel
  -> authorize by caller + target tool at tools/call
~~~

**Discovery is not permission to expose or execute.** Every newly discovered tool is disabled until its service owner approves it; a change to a tool description requires an explicit config review. This is especially important if the description is synthesized by a model: changes to tool semantics are a supply-chain input for agents.

At catalog scale, Uber uses an **incremental discovery surface**: `discover_server -> discover_tools -> get_tool_schema -> invoke_tool`. Coding agents instead commonly use Uber's `aifx mcp list/search/call` CLI and select file output to avoid loading huge responses into model context. A request-specific **response projection** can keep only specified nested fields from a larger tool result. Any projection must operate **after** the gateway's authorization and mandatory redaction; hiding a field from the model does not replace restricting it at the data source.

Prototype: consume a small OpenAPI or protobuf schema, derive three tools but register them disabled, then require a service-owner approval before discoverability or execution. Modify the source schema and assert the tool remains disabled or approval-pending until reviewed. Benchmark tool-context tokens and result byte size with/without discovery/projection, while asserting that direct forbidden `tools/call` remains blocked.

Uber reports 800+ servers and 5,000+ tools, but provides **no independent comparative proof** that its generator/gateway reduces security incidents. Source: https://www.uber.com/gp/en/blog/designing-mcp-gateway/ (2026-10-01).

## Failure modes and trade-offs

- **Single shared dependency:** a gateway outage or bad policy can disrupt all developers and agents. Use HA, deadlines, per-target circuit breakers and an audited break-glass path.
- **Target-side bypass:** access paths outside the gateway must have equivalent controls, or the gateway is only advisory.
- **Authorization vs discovery confusion:** hidden tools can still be invoked by name unless `tools/call` is enforced.
- **Manifest churn:** refresh cache/versioned schemas, test policy after tools change, detect removed/renamed operations.
- **Confused deputy risk:** never forward caller tokens blindly or let an agent choose arbitrary target credentials.
- **Audit/privacy:** log the decision, tool identity and result with redacted arguments rather than leaking user tokens or private payloads.
- **Vendor lock-in:** cross-account IAM, Cognito, AgentCore Identity, and DEFAULT/DYNAMIC are AWS-specific. In another environment use Entra/OIDC, a policy engine, a credential broker and remote MCP targets, then implement/test the missing protocol/session glue.

## Production evidence

Eon reports more than **100 developers** and around **10,000 sessions** through the gateway; developer onboarding fell from approximately **two days to two minutes**, and local MCP processes from about **ten to zero**. These are one customer's AWS-supported before/after observations, not independently audited reliability, security or agent-quality measurements.

## Prototype experiment

Expose three fake targets (Kubernetes read, Grafana read, GitHub comment write) behind a shared gateway. Test a human developer JWT, a read-only auditor bot, and an SRE bot; for each, enumerate tools and *directly attempt forbidden calls*. Rotate downstream credentials, kill one target, change a tool manifest mid-session, and inject a forged OAuth callback. Measure onboarding minutes, blocked-call correctness, number of stored secrets, median and p95 `tools/list`/`tools/call` latency, and audit coverage. Repeat with indexed and dynamic discovery on a synthetic 300-tool server.

## Sources

- Eon + AWS (2026-10-20), architecture and operational lessons: https://aws.amazon.com/blogs/machine-learning/how-eon-built-multi-cloud-investigation-with-an-mcp-gateway-on-amazon-bedrock-agentcore/
- AgentCore Gateway documentation: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway.html

## Related

- [Bind approvals to the exact side effect](enforcement-bound-agent-approvals.md)
- [Gradual tool disclosure with Spring AI](../spring/spring-ai-tool-search-advisor.md)
- [Contain coding agents at the environment boundary](environment-first-agent-containment.md)
- [Externalize authoritative agent telemetry](external-authoritative-agent-telemetry.md)
