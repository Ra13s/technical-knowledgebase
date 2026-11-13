# Persist ordered Spring AI message parts and turn-safe session state

## What it is

Spring AI **2.1.0-RC2** introduces ordered `MessagePart` content in messages and a structured event-log **Session Memory** API. For tool-using reasoning models, message text is not a faithful transcript: providers may require the exact ordering of thinking/reasoning, tool calls, tool results, redacted/signed reasoning, and opaque payload identifiers.

The key rule is **persist the complete ordered parts and session events, not just answer text**. Restore the provider-specific opaque metadata without editing or reordering it. The official documentation currently labels 2.1 as **release candidate/development**; stable 2.0.2 remains the production baseline unless the RC is deliberately piloted.

## Use when

- tool-calling, reasoning-enabled Spring AI applications must resume across application restarts;
- a model/API returns signed thought blocks or encrypted reasoning items;
- streaming tool responses must be aggregated without losing ordering/metadata;
- a multi-instance Spring service needs durable turn history, compaction and session identity.

## Implementation (2.1 RC pilot)

The session starter includes a JDBC repository, session service and advisors:

~~~xml
<dependency>
  <groupId>org.springframework.ai</groupId>
  <artifactId>spring-ai-starter-model-session-jdbc</artifactId>
</dependency>
~~~

Choose a Spring AI 2.1 RC BOM and a compatible Spring Boot version; use a development database first. For external PostgreSQL/MySQL schema creation during a local pilot:

~~~properties
spring.ai.session.repository.jdbc.initialize-schema=always
~~~

Manage schema migration explicitly in production; do not retain automatic creation without reviewing it.

~~~java
@Bean
ChatClient chatClient(ChatModel model, SessionService sessions) {
    return ChatClient.builder(model)
        .defaultAdvisors(SessionAdvisors.builder(sessions)
            .compaction(new TurnCountTrigger(20),
                WindowCompactionStrategy.builder().lastTurns(10).build())
            .build())
        .build();
}

// Every invocation needs a real, caller-scoped session id.
String answer = chatClient.prompt()
    .user("Continue the task")
    .advisors(SessionAdvisors.session("task-123"))
    .call()
    .content();
~~~

Session events have IDs, timestamps and metadata; compaction appends a view boundary and keeps the event log intact. It compacts **whole turns** so a tool call never loses its paired result. Never allow arbitrary callers to read another tenant's session merely by guessing an ID; enforce ownership in the service/repository layer.

Message content can be inspected as:

~~~java
ChatResponse response = chatModel.call(prompt);
AssistantMessage output = response.getResult().getOutput();
var parts = output.getParts();      // ordered, includes opaque provider payloads
var reasoning = output.getReasoning();
~~~

The parts model includes `TextPart`, `ReasoningPart`, `ToolCallPart`, `ToolResultPart`, `MediaPart` and `UnknownPart` (`org.springframework.ai.chat.messages.part`). Preserve provider attributes and opaque signatures on serialization; do not convert to a single `getText()` field for persisted history. Unknown parts are for same-provider pass-through, not permission to execute arbitrary embedded instructions.

## Migration and conformance test

In 2.1, old `ChatMemory`/repository and `MessageChatMemoryAdvisor` APIs are deprecated, planned for removal in 3.0; upgrade notes and migration guidance map them to Sessions. Existing historical chat-memory rows cannot reconstruct reasoning parts that were never stored.

Build a round-trip matrix:

1. Synchronous and streamed tool call: persist ordered parts -> reload -> return tool result; verify the provider accepts the continuation and no signature/id is lost.
2. Interrupt after receiving a tool call but before tool completion: resume with a coherent turn; use external [effect claims](../ai/durable-agent-tool-effects-and-fencing.md) to avoid repeating irreversible actions.
3. Compact 20+ turns; verify no half-turn and no dropped tool result; append-only raw log remains retrievable.
4. Restart on another application replica with JDBC session store; verify exact session ID ownership and prompt/history behavior.
5. Change providers mid-session; check text/tool-call interoperability, but **do not assume signed/encrypted reasoning transfers** across vendors.

## Caveats

- Signed thinking and encrypted payloads may be sensitive: encrypt at rest, enforce retention/access controls and avoid logging raw payloads.
- Preserving a conversation is not restoring its shell processes/filesystem or confirming external API effects.
- The old constructors work, but `getParts()` is the canonical order and `getToolCalls()`/`getMedia()` return unmodifiable views.
- One provider may reject replay metadata another accepts; provider-specific payloads should be replayed only to their origin.
- Session Memory 2.1 is not GA. Pin RC2 in an isolated branch, rerun integration tests after release, and avoid promising long-term API stability.

## Experiment

Prototype one Spring AI chat/tool service with JDBC Session Memory, a single reversible tool, one simulated crash before tool completion and a second worker continuing the same session. Measure replay-related HTTP 400s, reasoning/tool-part retention, repeated tool effects, compaction correctness, serialized payload size and p95 latency. Compare with the current 2.0.x text-only history on the same prompts.

## Sources

- https://spring.io/blog/2026/10/28/spring-ai-message-parts-reasoning-replay/
- https://docs.spring.io/spring-ai/reference/2.1/api/session.html
- https://docs.spring.io/spring-ai/reference/2.1/upgrade-notes.html
- https://docs.spring.io/spring-ai/reference/2.1/api/chatmodel.html

## Related

- [Durable agent effects and fencing](../ai/durable-agent-tool-effects-and-fencing.md)
- [Layered scoped agent memory](../ai/layered-scoped-agent-memory.md)
- [Structured output validation](spring-ai-structured-output-validation.md)
