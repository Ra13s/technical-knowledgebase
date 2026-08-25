# OpenAI Programmatic Tool Calling for bounded tool-heavy stages

## What it is

GPT-5.6 Programmatic Tool Calling (PTC) lets the model generate JavaScript that calls explicitly eligible tools inside a hosted V8 runtime, combines/reduces their outputs, and returns a smaller result to the model.

It removes a model round trip between every tool call when the intermediate work is mostly deterministic processing.

## Use when

Use PTC for a **bounded stage** where code can process many tool results without fresh model judgment after each call, for example:

- filtering;
- joining;
- ranking;
- deduplication;
- aggregation;
- deterministic validation;
- parallel lookups followed by a compact structured summary.

Keep tool calls direct when:

- one call is enough;
- intermediate outputs are already small;
- each result can change the model's next semantic decision;
- the action requires approval or has meaningful side effects;
- the final answer must preserve citations or provider-native artifacts exactly.

## How to use it with the OpenAI Agents SDK

A minimal Python example:

```python
from pydantic import BaseModel

from agents import Agent, ModelSettings, ProgrammaticToolCallingTool, Runner
from agents.decorators import tool


class InventoryOutput(BaseModel):
    sku: str
    available_units: int


@tool(allowed_callers=["programmatic"])
def get_inventory(sku: str) -> InventoryOutput:
    return InventoryOutput(sku=sku, available_units=42)


agent = Agent(
    name="Inventory planner",
    model="gpt-5.6",
    model_settings=ModelSettings(tool_choice="programmatic_tool_calling"),
    tools=[get_inventory, ProgrammaticToolCallingTool()],
)

result = Runner.run_sync(agent, "Check inventory for desk-lamp and summarize it.")
print(result.final_output)
```

Use:

```python
allowed_callers=["programmatic"]
```

for a tool that should only be callable from generated programs, or:

```python
allowed_callers=["direct", "programmatic"]
```

when both routes are valid.

## Harness rule

Do not merely expose PTC and tell the model to "use it efficiently". Give it an explicit stage boundary:

```text
Use Programmatic Tool Calling only for the bounded lookup-and-reduction stage.
It may call inventory_lookup and pricing_lookup.
Run independent calls concurrently when safe.
Return exactly: {sku, available, price, evidence_ids}.
Retry transient failures at most once and stop after all requested SKUs are resolved.
Do not perform side-effecting actions.
Use direct tool calls for approval, semantic judgment, and final validation.
```

Document eligible tools' return fields, types, and error behavior. If the model cannot know the return shape before writing the generated program, prefer direct calls so it can inspect the result first.

## Why it is useful

- Avoids unnecessary model turns for deterministic processing between tool calls.
- Can reduce large intermediate tool outputs before they enter model context.
- Supports loops, branching, parallel calls, and calculations in the generated program.
- Tool access is explicit: the hosted runtime can only call tools opted in for programmatic access.

## Validation rule

Benchmark PTC against the existing direct-tool path on representative tasks. Compare:

- task success;
- final-answer completeness;
- required evidence/citations;
- total tokens;
- latency;
- cost;
- tool calls, turns, retries.

Do not accept "fewer calls" as a win if the final answer loses evidence or required fields.

Also validate **both** outputs: the generated `program_output` can be correct while the final assistant message still omits a required field or caveat.

## Caveats / when not to use

- PTC requires a supported OpenAI Responses model; GPT-5.6 is the documented baseline. It is not a Chat Completions feature.
- The Agents SDK allows at most one `ProgrammaticToolCallingTool()` on an agent.
- The hosted V8 program has no Node.js APIs, filesystem, arbitrary network access, or persistent process. It interacts through eligible tools only.
- Approval-sensitive/high-impact tools should normally remain direct calls. In the Agents SDK, existing tool guardrails, timeouts, approvals, and hooks still apply to program-owned calls.
- Preserve caller/call linkage when implementing the raw Responses API loop: `program`, child tool calls, and `program_output` are separate items.
- Treat the feature as an orchestration optimization, not a reason to move semantic decision-making into generated code.

## Version / compatibility

Verified against current GPT-5.6 model guidance and the OpenAI Agents SDK Python documentation. PTC and `allowed_callers` are Responses-only capabilities and should be rechecked when changing model/API families.

## Sources

- https://developers.openai.com/api/docs/guides/latest-model
- https://openai.github.io/openai-agents-python/tools/

## Related

- [`Spring AI ToolSearchToolCallingAdvisor`](../spring/spring-ai-tool-search-advisor.md) — solves large tool-catalog disclosure; PTC solves deterministic orchestration/reduction after tools are available.