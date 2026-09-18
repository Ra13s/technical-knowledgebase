# Encode organizational workflows as executable agent playbooks

## What it is

A reusable agent-workflow pattern that turns institutional engineering knowledge into **versioned executable capabilities** instead of leaving it in static documentation, giant system prompts, or senior engineers' heads.

A playbook packages enough information to execute one recurring workflow consistently: purpose, inputs, context references, agentic and deterministic steps, tools, permissions, validation, expected outputs, and safety boundaries. The agent discovers and invokes the playbook when relevant rather than loading every workflow into every prompt.

LinkedIn's CAPT and DoorDash's Flux independently use this model at production scale. Spring AI Community's Agent Skills provides a smaller Java/Spring implementation of the same progressive-disclosure idea for reusable `SKILL.md` modules.

## Use when

Use playbooks when a task is repeated often enough that engineers already have a recognizable procedure, especially when it spans several systems or requires domain knowledge:

- incident or customer-issue triage;
- experiment/feature-flag cleanup;
- service or endpoint creation;
- code-review or remediation workflows;
- CI/on-call/maintenance procedures;
- migrations that combine APIs, code search, tests and deployment checks.

Do not create a playbook for a one-off task whose steps are not yet understood. First observe the workflow and identify the stable contract.

## Playbook contract

A useful playbook is more than prose. Treat it as a deployable workflow artifact.

```yaml
name: cleanup-experiment
purpose: Remove a completed experiment and make the winning behavior permanent
inputs:
  experiment_id: string
context:
  - experiment metadata
  - repository code
steps:
  - agent: inspect experiment result and affected code paths
  - deterministic: assert experiment is finalized
  - agent: remove losing branches and stale flags
  - deterministic: run targeted tests
  - agent: summarize changed behavior
capabilities:
  tools:
    - experiment.read
    - code.search
    - git.edit
    - test.run
  network:
    - experiments.internal
validation:
  - no stale flag references remain
  - targeted tests pass
outputs:
  - patch
  - evidence summary
safety:
  requires_human_merge: true
```

The exact format is implementation-specific. The important part is that workflow, capability and validation requirements are explicit enough for the surrounding platform to enforce them.

## Implementation rules

### 1. Keep playbooks small and composable

Prefer focused building blocks over a single "engineering everything" workflow. LinkedIn exposes playbooks as tools so agents can select and chain them dynamically.

A good boundary is one task with a clear trigger, inputs, outputs and verification rule.

### 2. Separate agentic judgement from deterministic steps

Put flexible interpretation, diagnosis and code changes in agentic steps. Put predicates, schema checks, test commands, approvals and other hard gates in conventional code.

```text
agent inspect -> deterministic eligibility check -> agent change -> deterministic validation
```

DoorDash explicitly allows playbooks to mix both forms so logic can move between agent reasoning and deterministic execution as the workflow matures.

### 3. Discover capabilities on demand

Do not preload hundreds of playbooks and tool schemas into every model request.

Use a progressive-disclosure surface such as:

```text
list/search capability metadata
  -> inspect one relevant schema/playbook
  -> execute selected capability
```

LinkedIn replaced large MCP tool catalogs with three meta-tools for discovery, schema inspection and execution. The stable model-facing surface remains small even as the underlying catalog grows.

Spring AI Community's `SkillsTool` follows the same principle: discover a skill by name/description first, then load the full `SKILL.md` only when the task matches.

### 4. Derive permissions from the workflow contract

A playbook that declares `deploy.read` should not receive arbitrary deployment write access "because the agent may need it".

```text
playbook capabilities
        |
        v
policy / gateway
        |
        +-- scoped identity
        +-- allowed tools
        +-- egress policy
        +-- audit log
```

DoorDash's Flux has each playbook declare its required tools and grants scoped permissions through a centralized MCP gateway. Treat the manifest as input to enforcement, not as a prompt that asks the model to self-restrict.

### 5. Support central and local workflow ownership

Use two scopes when the organization needs both shared standards and repository/domain specialization:

- **central playbooks** for cross-cutting workflows and policy;
- **local playbooks** for repository/service-specific operations.

Resolve both into one discoverable catalog. This lets domain experts improve local workflows without turning the central platform team into a bottleneck.

### 6. Keep invocation separate from workflow definition

The same playbook should be callable from the surface that best matches the trigger: CLI, IDE, Slack, GitHub, cron, ticket event, or another agent.

```text
Slack ----+
GitHub ---+
cron -----+--> playbook --> sandbox/tools --> evidence/output
CLI ------+
```

This keeps business workflow semantics from being duplicated across chat bots, CI jobs and local tools.

### 7. Instrument playbooks as products

Record at least:

- playbook/version;
- invocation source;
- tools/capabilities used;
- success/failure and validation result;
- latency and model/tool cost;
- human correction or override;
- output acceptance/merge where meaningful.

Use this telemetry to retire low-value playbooks, find brittle steps and identify repeated tool sequences worth encoding as new workflows.

## Spring AI implementation option

For a smaller Spring-based agent, `spring-ai-agent-utils` currently provides `SkillsTool` from the Spring AI Community project.

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springaicommunity</groupId>
      <artifactId>spring-ai-agent-utils-bom</artifactId>
      <version>0.12.0</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependency>
  <groupId>org.springaicommunity</groupId>
  <artifactId>spring-ai-agent-utils</artifactId>
</dependency>
```

Register reusable skills:

```java
var skills = SkillsTool.builder()
    .addSkillsDirectory(".claude/skills")
    .build();

ChatClient client = chatClientBuilder
    .defaultTools(skills)
    .build();
```

A skill directory contains `SKILL.md` with YAML metadata plus instructions and may include scripts, references and assets. Use this as an implementation primitive for portable knowledge modules, not as a substitute for a real permission/sandbox layer.

At the 2026-09 intake, the community repository documents Java 17+, Spring Boot 3/4 and Spring AI 2.0+; verify the current version before pinning.

## Why it is useful

The core shift is from **documentation as context** to **workflow as executable organizational knowledge**.

A mature playbook can encode the route an experienced engineer would take through internal systems while still leaving hard permissions, tests and approvals to deterministic infrastructure. It also survives model changes better than a monolithic prompt because the task contract and integrations remain explicit.

LinkedIn reports more than 500 playbooks used by more than 1,000 engineers, with about 70% lower initial triage time in many areas and roughly 3x faster common data-analysis workflows. DoorDash reports more than 300 playbooks and over 10,000 weekly playbook invocations. These are company-specific outcomes, not transferable benchmarks, but they demonstrate that the pattern can scale beyond a prototype.

## Failure modes and caveats

- **Playbook sprawl:** hundreds of weak or overlapping playbooks become another search problem. Add ownership, tags, telemetry and retirement rules.
- **Stale institutional knowledge:** a workflow can be executable and still wrong. Version dependencies and validate assumptions against live systems.
- **Prompt-only permissions:** declaring allowed tools in Markdown does not enforce them. Enforce capabilities at the gateway/sandbox boundary.
- **Hidden unsafe scripts:** a skill that bundles shell scripts expands the execution surface. Review/pin it like code and run it inside the normal sandbox.
- **Central bottleneck:** requiring a platform team to author every domain workflow prevents the catalog from compounding.
- **Over-agentification:** deterministic operations should stay deterministic when no judgement is needed.
- `spring-ai-agent-utils` is a Spring AI Community project, not the Spring AI core API; treat its compatibility and security model accordingly.

## Prototype experiment

Pick one recurring engineering workflow that currently requires a checklist plus several tools, for example failed-deployment triage.

1. record the current expert workflow and evidence sources;
2. encode a playbook with explicit inputs, tools, permissions, deterministic checks and outputs;
3. expose only metadata initially and load the full workflow on demand;
4. route its declared capabilities through the existing sandbox/gateway boundary;
5. invoke the same playbook from both CLI and an issue/chat trigger;
6. compare time-to-result, tool-call count, human corrections, failures and cost with an unstructured agent prompt;
7. promote only if the playbook improves repeatability without hiding important judgement.

## Sources

- LinkedIn Engineering, *Contextual agent playbooks and tools: How LinkedIn gave AI coding agents organizational context* (2026-01-27): https://www.linkedin.com/blog/engineering/ai/contextual-agent-playbooks-and-tools-how-linkedin-gave-ai-coding-agents-organizational-context
- DoorDash Engineering, *Delegating Engineering Work To Cloud-Based Agents* (2026-08-11): https://careersatdoordash.com/blog/delegating-engineering-work-to-cloud-based-agents/
- Spring, *Spring AI Agentic Patterns (Part 1): Agent Skills* (2026-01-13): https://spring.io/blog/2026/01/13/spring-ai-generic-agent-skills/
- Spring AI Community, `spring-ai-agent-utils`: https://github.com/spring-ai-community/spring-ai-agent-utils

## Related

- [Resolve repository instructions by target path](path-scoped-repository-instructions.md)
- [Contain coding agents at the environment boundary first](environment-first-agent-containment.md)
- [Spring AI ToolSearchToolCallingAdvisor](../spring/spring-ai-tool-search-advisor.md)
- [OpenAI Programmatic Tool Calling](openai-programmatic-tool-calling.md)
