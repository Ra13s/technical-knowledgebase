# Scope Spring AI Redis semantic cache by authenticated caller

## What it is

In Spring AI **2.1.0 GA / 2.0.3 GA (2026-11-20)** the Redis SemanticCacheAdvisor requires a SemanticCacheScopeResolver. The resolver determines which callers may reuse model outputs. Without isolation, a semantic cache can return private answers to a different user whose question is similar. This release fixes **CVE-2026-59417** (cross-user disclosure/cache poisoning); do not keep affected older versions in production.

This is a concrete **authorization and retention** boundary for a probabilistic cache, not simply a performance configuration.

## Use when

- Caching responses based on embedding similarity in a shared Redis instance.
- One app serves multiple users, accounts or tenants.
- Answers are conditioned on private documents, tools, account state, system prompts or entitlement sets.
- Migrating Spring AI 2.0.x/2.1 RC applications with Redis semantic caching.

## Implementation

Use a supported Spring AI baseline (2.1.0 with Spring Boot 4.2, or 2.0.3 on its compatible Boot line). The 2.1 documentation lists these components for Redis caching, with Spring AI BOM managing versions:

~~~xml
<dependency>
  <groupId>org.springframework.ai</groupId>
  <artifactId>spring-ai-autoconfigure-vector-store-redis-semantic-cache</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.ai</groupId>
  <artifactId>spring-ai-redis-semantic-cache</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.ai</groupId>
  <artifactId>spring-ai-transformers</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
~~~

Redis auto-configuration expects a Jedis connection factory; set spring.data.redis.client-type=jedis if using this implementation (Boot otherwise defaults to Lettuce).

Build the advisor with an explicit resolver and TTL:

~~~java
@Bean
SemanticCacheAdvisor semanticCacheAdvisor(SemanticCache cache) {
    return SemanticCacheAdvisor.builder()
        .cache(cache)
        .scopeResolver(request -> (String) request.context().get("trustedCacheScope"))
        .ttl(Duration.ofHours(1))
        .build();
}
~~~

**Caller-supplied headers/fields are not trustedCacheScope.** Derive the scope from authenticated identity in the request-handling layer and pass it into ChatClient context. If a session includes unique private data or changing permissions, use a per-user/per-entitlement scope or bypass caching rather than using merely a tenant name. If the scope is null or blank, Spring AI's advisor bypasses cache read and write. A constant such as shared is suitable only for truly public responses.

The resolver may be called on another thread for streaming; prefer ChatClient request context over ThreadLocal/SecurityContextHolder.

~~~java
String scope = verifiedPrincipal.getTenantId() + ":" + verifiedPrincipal.getUserId();
// Consider adding entitlement or data-snapshot version for private/RAG-derived answers.
ChatResponse answer = chatClient.prompt()
    .user(question)
    .advisors(a -> a.param("trustedCacheScope", scope))
    .call()
    .chatResponse();
~~~

The example is illustrative; choose a collision-safe opaque scope and protect its values in logs.

## Hardening checks

- Ensure scope is generated from **verified** identities, not from request parameters that users can forge.
- Default to no cache on missing identity/scope; do not silently fall back to a global cache.
- Choose a finite TTL; the advisor has **no expiration by default**. TTL support requires a RedisVectorStore-backed DefaultSemanticCache or storing can fail with UnsupportedOperationException.
- Make cache invalidation cover logout/role changes, document access changes, prompt/template changes and sensitive data revocation. Version scopes or purge entries when appropriate.
- Set sensitivity-appropriate similarity thresholds and test false matches, not just cache-hit rate.
- A direct SemanticCache.get/set without the advisor's context hash may bypass the scope-based isolation; restrict/encapsulate direct usage and test it separately.
- Investigate deployment-specific cache key collisions and embeddings leakage. Do not assume semantic equivalence establishes that two users are authorized for the same answer.

## Test matrix

With Redis Testcontainers and a fake ChatModel counting invocations, test two principals A/B with semantically identical private questions. A's answer must never be returned to B, even if its embedding is closer. A repeated authorized query may hit its scope. A forged scope header must not change the result. Blank scope must not use the cache. Streamed and synchronous queries must behave identically. After role change or sensitive document revocation, changed scope must miss. Inject concurrent A/B requests and instrument cross-scope cache hits as a fail-the-build assertion.

## Version and security notes

Spring AI 2.1.0 fixes 13 CVEs accumulated over milestones and preceding releases; 2.0.3 carries fixes for its line. Other behavior-changing fixes include Typesense filter literal parsing and Tika extracted-text limits. Audit release notes before upgrading, especially when application code relied on permissive cache sharing, Typesense filter syntax or very large documents.

## Experiment

Take a real RAG endpoint on a test database, generate 100 repeated and cross-tenant semantically overlapping prompts, rotate one user's permissions mid-run and try forged request context. Measure leaked responses (target zero), missed invalidations (target zero), cache-hit rate, p95 latency and Redis cardinality with per-user vs per-tenant scoping. Treat a nonzero exposure as a release blocker regardless of hit-rate gain.

## Sources

- Spring AI 2.1 GA + 2.0.3 GA (2026-11-20): https://spring.io/blog/2026/11/20/spring-ai-2-1-0-GA-available-now/
- Spring AI Redis semantic cache reference: https://docs.spring.io/spring-ai/reference/2.1-SNAPSHOT/rag/semantic-cache.html
- Spring AI upgrade notes: https://docs.spring.io/spring-ai/reference/2.1-SNAPSHOT/upgrading/spring-ai-2-0.html

## Related

- [Spring AI ordered session memory](spring-ai-session-parts-replay.md)
- [Identity-scoped MCP gateway](../ai/identity-scoped-mcp-gateway.md)
- [MCP protocol upgrades and conformance](../ai/mcp-protocol-upgrades-and-conformance.md)
