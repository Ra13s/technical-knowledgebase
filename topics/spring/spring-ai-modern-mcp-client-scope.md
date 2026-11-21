# Isolate Spring AI 2.1 modern MCP tool clients by caller

## What it is

Spring AI **2.1.0 GA (2026-11-20)** supports the stateless MCP 2026-07-28 revision alongside legacy clients. A modern MCP client has no initialize/session handshake, so tool discovery and invocations can be bound to a *particular caller* when registered correctly. Critically, a multi-user tool catalog or authorization token must not escape from one user's request into another's.

MCP Java SDK **2.2.0+** is required. Currently the property-configured modern client uses JDK HttpClient with Streamable HTTP; other transports need a programmatic transport bean. Do not confuse modern MCP classes with legacy namesakes.

## Use when

- A Spring Boot ChatClient consumes MCP tools from multiple services, including caller-specific or privileged tools.
- Different tenants/users see different tool listings or use distinct downstream identities.
- Migrating legacy MCP tools incrementally to the stateless revision without a flag day.
- WebFlux apps need tool discovery without blocking event-loop threads.

## Implementation recipe

A mixed-protocol client:

~~~yaml
spring:
  ai:
    mcp:
      client:
        streamable-http:
          connections:
            billing:
              url: http://localhost:8081  # legacy by default
            tickets:
              url: http://localhost:8080
              protocol: MODERN
              client-scope: PER_REQUEST
              listen: false
        request-timeout: 20s
~~~

For modern connections, PER_REQUEST is the default and resolves tools independently for each ChatClient request. Use SHARED **only** if every request goes to the server under one app/service identity with a genuinely common tool listing. For per-user caching, use CUSTOM and a SyncModernMcpClientScope / AsyncModernMcpClientScope that keys the client by authenticated principal and environment, handles eviction/revocation and does not pool across tenants.

Bind callbacks at request time:

~~~java
@Bean
ChatClient chatClient(ChatModel model, ToolCallbackProvider mcpTools) {
    return ChatClient.builder(model)
        .defaultToolCallbacks(mcpTools)
        .build();
}
~~~

**Do not** take mcpTools.getToolCallbacks() once at application startup and share that snapshot: callbacks can be bound to the first/no caller's client. Do not re-expose modern downstream tools with spring.ai.mcp.server.expose-mcp-client-tools=true; those are also resolved outside of caller context. If building an MCP gateway, resolve downstream tools inside each authenticated server request instead.

For caller identity, use a McpClientCustomizer to set the client's transportContextProvider; an McpHttpRequestCustomizer converts the transport context into HTTP headers. Carry authenticated identity, **not user-supplied role headers**, and let the downstream MCP server independently enforce authorization at tools/call, not only at tools/list.

## Cache and streaming traps

- A modern server may return tools/list cache hints ttlMs and cacheScope. A public listing can be shared; a private one must remain caller-scoped. A SHARED client can accidentally retain the first caller's private tool list for others.
- listen=true opens an application-wide stream, not a user stream. It needs application credentials or no credentials; do not silently use whichever caller happened to initialize it.
- The ToolCallbackProvider contract blocks during tool listing even for async MCP clients. In WebFlux, start ChatClient on boundedElastic if list resolution may block:

~~~java
Flux<String> answer = Flux.defer(() -> chatClient.prompt(question).stream().content())
    .subscribeOn(Schedulers.boundedElastic());
~~~

- Per-request discovery costs a tools/list round-trip unless public caching applies. This is the correct price for isolation when tools vary by caller.
- Modern MCP has no automatic session memory. Conversation/session persistence and side-effect recovery are separate concerns.

## Verification matrix

1. Legacy server and MODERN server coexist; verify both tool sets appear and no invented cross-protocol behavior.
2. Tenant A has tool A-only; B has B-only. Alternate requests A -> B -> A, including one tool-list TTL > 0; assert no cross-tenant listings or invocation credentials.
3. Test unauthenticated tool calls and direct requests to tools/call. A hidden tool must still deny unauthorized invocation.
4. Rotate/revoke A's credentials while requests are running; verify new requests deny and that CUSTOM-cached clients are invalidated.
5. Kill the modern server; confirm unreachable cool-down omits its tools without suppressing healthy services. On recovery, refresh.
6. Run ChatClient from WebFlux request threads; assert no blocking-thread IllegalStateException after offload.
7. Listen for tool changes on a single-identity server; verify invalidated catalog and that listen-stream auth does not borrow a user's token.

## Caveats and limits

The protocol is **not** an authorization mechanism. Spring AI's context propagation helps associate identity, but the MCP server/gateway must authorize each tool call. Do not enable SHARED solely for latency if calls are user-specific. Check SDK transport/version compatibility and upgrade notes for server capabilities such as elicitation/progress.

## Experiment

Run two test MCP servers (a public knowledge server and a tenant-private ticket server) under Spring AI 2.1/Boot 4.2. Compare PER_REQUEST vs a deliberately misconfigured SHARED client, then CUSTOM with per-user eviction. Test 100 interleaved authenticated clients with private/public cached listings and changing entitlements. Record cross-tenant exposure (must be zero), p95 listing latency, cache hit rate and revoked-call denial.

## Sources

- Spring AI 2.1 GA (2026-11-20): https://spring.io/blog/2026/11/20/spring-ai-2-1-0-GA-available-now/
- MCP 2026-07-28 modern connections, Spring AI 2.1 reference: https://docs.spring.io/spring-ai/reference/2.1-SNAPSHOT/mcp/client/modern.html
- MCP client Boot starter: https://docs.spring.io/spring-ai/reference/api/mcp/mcp-client-boot-starter-docs.html

## Related

- [MCP protocol upgrades and conformance](../ai/mcp-protocol-upgrades-and-conformance.md)
- [Identity-scoped MCP gateway](../ai/identity-scoped-mcp-gateway.md)
- [Spring AI ordered sessions](spring-ai-session-parts-replay.md)
