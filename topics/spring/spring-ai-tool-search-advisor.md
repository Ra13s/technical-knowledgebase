# Spring AI `ToolSearchToolCallingAdvisor`

## What it is

Spring AI 2.0 can avoid sending every registered tool definition to the model on every request. `ToolSearchToolCallingAdvisor` indexes the tool catalog per session and initially exposes only a tool-search capability; matching tool definitions are added when the model asks for them.

## Use when

Evaluate it when a `ChatClient` has:

- about 10+ tools
- tool definitions consuming roughly 10K+ context tokens
- several MCP servers contributing tools
- degraded tool-selection accuracy because the catalog is large

For a small tool set where most tools are used frequently, keep the default `ToolCallingAdvisor`; the extra discovery round-trip is usually unnecessary.

## Spring Boot setup

Add the starter:

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-tool-search-advisor</artifactId>
</dependency>
```

Enable it:

```properties
spring.ai.chat.client.tool-search-advisor.enabled=true
spring.ai.chat.client.tool-search-advisor.tool-index-type=regex
```

`regex` is the default and needs no extra index infrastructure. Alternatives are:

- `lucene` — keyword search
- `vector` — semantic search; requires a `VectorStore`

When enabled, Spring Boot replaces the default tool-calling advisor with the tool-search advisor.

## Session ID is required

The tool index is scoped per conversation/session. Pass a stable conversation ID on requests:

```java
String answer = chatClient.prompt()
    .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, "user-42-session"))
    .user("...")
    .call()
    .content();
```

If the application already uses `MessageChatMemoryAdvisor` with `ChatMemory.CONVERSATION_ID`, the same context key can scope tool discovery.

## Useful tuning

Limit how many definitions a discovery search can expose:

```properties
spring.ai.chat.client.tool-search-advisor.max-results=5
```

Choose the simplest index that retrieves tools reliably:

1. start with `regex` for small, clearly named catalogs;
2. use `lucene` when keyword relevance is useful;
3. use `vector` when descriptions/intent matter more than names and a vector store already exists.

## Why it is useful

Progressive tool disclosure keeps a large tool catalog out of the model context until it is needed. This is especially useful for MCP-heavy agents where dozens of verbose tool schemas otherwise consume context before the task starts.

## Caveats

- Requires **Spring AI 2.0** APIs/configuration shown here.
- Tool discovery adds another model/tool-search step, so it is not automatically better for small catalogs.
- Session IDs matter: missing or incorrectly shared IDs can break conversation isolation and tool-index lifecycle assumptions.
- Measure both token use and tool-selection correctness; reducing prompt size is not useful if retrieval hides the tool the model needs.

## Sources

- https://docs.spring.io/spring-ai/reference/api/tools.html
- https://docs.spring.io/spring-ai/reference/2.0-SNAPSHOT/api/tools/tool-search-tool.html
- https://spring.io/blog/2026/06/15/spring-ai-composable-tool-calling/
- https://spring.io/blog/2026/06/12/spring-ai-2-0-0-GA-available-now/
