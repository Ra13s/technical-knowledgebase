# Event-triggered infrastructure agents with plan-and-policy PR gates

## What it is

A production-infrastructure agent may investigate a live incident with read-only telemetry and resource graph context, but it should **propose infrastructure-as-code changes through a versioned PR and deterministic preview/policy gate**, rather than applying production mutations from the reasoning loop. Pulumi's November 18, 2026 Agents + Delivery design exposes this separation as a platform pattern: event/schedule-triggered agents, their own identities/roles, IaC proposals, preview plans, and delivery approval policies.

This is an implementable **incident -> diagnosis -> change proposal -> reviewed deploy** loop, not a claim that any AI verdict is sufficient to authorize an apply. Pulumi Agents and Delivery launched in public preview; role and feature availability vary by edition.

## Use when

- Cloud/Kubernetes drift, resource pressure, failed deployment or incident alerts produce repeatable infrastructure fixes.
- Observability and runbooks are already available, but engineers still correlate them manually to IaC and stack state.
- Agents must investigate in production without inheriting an engineer's broad credentials.
- A proposed remediation can be expressed as a small auditable change to a controlled repo.

## Build the workflow

~~~text
Datadog/PagerDuty/Sentry/Honeycomb/stack drift event
  -> validate signed event + dedupe alert fingerprint
  -> create run record (source, environment, incident, base SHA, time)
  -> read-only incident agent, isolated workspace
       -> fetch runbook + metrics/logs + resource graph + deployment history
       -> identify evidence-backed candidate and confidence
  -> tightly bounded IaC edit on one worktree/branch
  -> pulumi preview --diff (or Terraform plan) with pinned state and stack
  -> reject out-of-scope, destructive, policy-denied or drifted plans
  -> immutable plan summary + evidence -> PR
  -> normal CI + independent reviewer / human approval on risk
  -> trusted deployment workflow revalidates live state, plan and policy
  -> canary/rollout and alert feedback linked to incident
~~~

Example **logical configuration sketch**, not Pulumi's actual agent schema:

~~~yaml
agent:
  name: incident-memory-investigator
  trigger: datadog.monitor.alert
  target: service/checkout
  principal: sre-investigator
  permissions:
    cloud: read-only
    telemetry: read-only
    repository: branch-and-pr-only
  output:
    - incident-evidence.json
    - proposed-change.diff
    - preview-plan.json
  limits:
    max_repair_attempts: 2
    max_runtime_minutes: 15
  deployment: never-direct
~~~

The Pulumi product lets teams define agents in code with role, instructions and integrations; this YAML describes the **transferable contract**, not a paste-ready Pulumi config. In a trial, configure actual agent and integration permissions via the Pulumi docs and require a protected deployment pipeline. The agent's Pulumi Cloud role limits stacks/ESC environments, **while connected-tool credentials separately determine actual privileges**. Merely selecting an integration or writing “read-only” in instructions does not restrict that token.

## Authority and state boundaries

1. **Event ingestion:** require a source event ID and deterministic idempotency key; discard replay or attach it to an existing active run. Alert text is untrusted input.
2. **Runtime isolation:** agent cannot call production apply APIs or obtain deploy credentials; connected telemetry/stack credentials are intentionally read-only. Assign its own service identity and revoke independently of engineers.
3. **Human-vs-agent permission intersection:** for shared tasks, check the teammate's permission *and* the agent role; approvals cannot grant permissions outside either credential's scope. Pulumi's default Auto mode does not itself mean actions are safe.
4. **Stable context:** capture repo/base/head SHA, Pulumi stack, target environment, event ID, observation window, runbook version and external tool-call IDs. Avoid stale-context remediations.
5. **Deterministic gate:** run syntax, IaC/unit tests, preview, policy-as-code and secret checks; inspect resources added/changed/deleted and require manual review for replacement, public ingress, IAM broadening, destructive changes and uncertain drift.
6. **Apply gate:** a separate deployment principal rechecks current stack state and required approvals immediately before apply; use protected promotion and rollout, not the agent's earlier assessment.
7. **Reconciliation:** capture plan-to-apply difference, alert recovery, rollback/incident regression and human repair effort. The agent reporting “fixed” is not a postcondition.

An agent-provided “deployment risk score” can prioritize reviewers, but cannot override policy-as-code or safety rules. If a plan changed since review, invalidate approval or require a new one.

## Evidence and limitations

Pulumi describes event-triggered incident, drift and security agents, separate agent roles, source-defined agents and delivery stages. It also claims that the share of infrastructure operations performed by agents on its platform grew from 0% in January to over 40% in October 2026. This is vendor-reported usage, **not** a controlled measure of safer operations or fewer incidents. The initial product is public preview, and “never change production directly” describes the intended Agents-to-Delivery path, not a blanket restriction on all external integrations' credentials.

This pattern overlaps with the KB's deterministic outer loop and identity-scoped gateways, but adds **live deployment-state correlation, a concrete IaC plan artifact and post-deploy verification**.

## Failure modes

- Alert storms create duplicate PRs; coalesce by service/symptom and preserve provenance.
- A read-only agent whose Datadog/Pulumi token actually has write scopes can still mutate external systems.
- A valid preview is not an immutable approval when the stack has changed since the preview.
- A “small” memory adjustment can mask a leak or induce cost/performance regressions; require diagnosis, not only knob turning.
- IaC is not the only path to production change; ensure manual consoles and privileged CLIs have equivalent governance.
- Sensitive production logs or runbooks may contain prompt injections or secrets: sanitize and keep least-privileged execution.

## Prototype experiment

Create a fake monitor for a Kubernetes Deployment with synthetic OOM events and seeded misleading alerts. Give an investigator read-only access to metrics, logs and stack info; allow it to propose a memory-limit patch in a toy IaC repository. A trusted job runs preview + OPA/Pulumi policy and posts a PR. Inject: duplicate alert, forged alert payload, stale stack revision, broad IAM change and a second unrelated resource modification. Require zero unauthorized operations; count correct vs unnecessary PRs, plan drift, human review minutes, time-to-safe-proposal and false diagnosis. Compare with an engineer following the runbook.

## Sources

- Pulumi, Introducing Pulumi Agents (2026-11-18): https://www.pulumi.com/blog/introducing-pulumi-agents/
- Pulumi, Everything after code, as code (2026-11-18): https://www.pulumi.com/blog/everything-after-code-as-code/
- Pulumi Agents docs: https://www.pulumi.com/docs/ai/agents/
- Pulumi permissions docs: https://www.pulumi.com/docs/ai/agents/permissions/
- Pulumi integrations docs: https://www.pulumi.com/docs/ai/agents/integrations/
- Independent launch reporting (not an efficacy study): https://www.geekwire.com/2026/absurd-ambition-pulumi-expands-into-software-delivery-with-ai-agents-that-ship-and-fix-code/

## Related

- [Deterministic outer loop](deterministic-outer-loop-agent-platform.md)
- [Identity-scoped MCP gateway](identity-scoped-mcp-gateway.md)
- [Operation-level agent sandbox policy](operation-level-agent-sandbox-policy.md)
- [Event-reducer orchestration](event-reducer-coding-agent-orchestration.md)
